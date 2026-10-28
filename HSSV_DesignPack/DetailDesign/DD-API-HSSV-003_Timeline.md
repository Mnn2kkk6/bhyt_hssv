# Thiết kế chi tiết — API xem timeline thẻ và gián đoạn

| Mục | Nội dung |
|---|---|
| Tên dự án | Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200M |
| Hệ thống | Hệ thống tra cứu kết quả BHYT HSSV |
| Tài liệu | Thiết kế chi tiết (Detail Design) |
| Mã tài liệu | DD-API-HSSV-003 |
| Phiên bản / trạng thái | 1.0 / Draft – chờ review |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Tài liệu cơ bản | `HSSV-BD-001` |
| Chức năng | F-HSSV-004 / API-HSSV-003 |
| Nguồn | Repo `https://github.com/Mnn2kkk6/bhyt_hssv`; quy tắc tuần 1–2; template Detail Design đã cung cấp |

## 1. Tổng quan module

| Mục | Nội dung |
|---|---|
| ID module | DD-API-HSSV-003 |
| Hiện thực cho | F-HSSV-004 / API-HSSV-003; Basic Design `HSSV-BD-001` |
| Mức chi tiết | B – API nghiệp vụ / A – batch quy mô lớn |
| Tóm tắt xử lý | Resolve published snapshot → xác nhận quyền và người có kết quả → truy vấn timeline theo trang → truy vấn gaps đã tính → đối soát tổng ngày → trả response. |
| Lớp/package đề xuất | `HssvController` → `HssvQueryService` / `HssvExportService` → `HssvReadRepository` / `ExportJobRepository` |
| Chữ ký phương thức | `PersonTimelineResponse getTimeline(String soSoBhxh, String snapshotId, PageRequest pageRequest)` |
| Được gọi bởi | API Gateway / batch orchestrator |
| Gọi tới | PostgreSQL read model, Redis cache nếu phù hợp, Object Storage nếu cần |
| Giao dịch | Đọc dùng read-only transaction; thao tác tạo job dùng transaction ngắn; không giữ transaction trong lúc tạo file |
| Xử lý đồng thời | Snapshot chỉ đọc ở trạng thái `PUBLISHED`; keyset/sắp xếp ổn định; export job có trạng thái và khóa idempotency |

## 2. Định nghĩa vào/ra

| Hướng | No | Tên tham số / thuộc tính | Kiểu | Bắt buộc | Mô tả / miền giá trị | Giá trị khi không có | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| IN | 1 | soSoBhxh | String path | Có | Mã BHXH của người thuộc tập kết quả. | — | API-HSSV-003 |
| IN | 2 | snapshotId | String query | Không | Snapshot PUBLISHED. | Latest PUBLISHED | BD §6 |
| IN | 3 | page/pageSize | Integer | Không | Phân trang timeline; pageSize 1..100. | 1 / 100 | BD §8 |
| OUT | 1 | windowStart/windowEnd | Date | Có | Cửa sổ cá nhân được dùng để tính gián đoạn, nếu thuộc GAP_90_PLUS. | null với STOPPED nếu không tính | BD §8 |
| OUT | 2 | cards[] | Array | Có | Thẻ theo thời gian; có status và isValid. | [] không kỳ vọng nếu person result có timeline | BD §8 |
| OUT | 3 | gaps[] | Array | Có | Các khoảng trống tính theo ngày inclusive. | [] nếu không có gap | BD §8 |
| OUT | 4 | totalGapDays/gapCount | Integer | Có | Tổng và số khoảng gap đã kiểm chứng. | 0 | BD §8 |

## 3. Xử lý chi tiết

| Bước | Nội dung xử lý | Điều kiện | Khi đúng | Khi sai | SQL / Check / MSG-ID | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Xác thực quyền `HSSV_RESULT_READ_TIMELINE` và validate ID, snapshot, pagination. | Hợp lệ? | Sang bước 2. | Trả 401/403/400. | CHK-01, CHK-02 |  |
| 2 | Resolve snapshot PUBLISHED và kiểm tra người thuộc tập kết quả snapshot. | Có result row? | Sang bước 3. | 404 hoặc 409 theo snapshot. | SQL-01, SQL-02 |  |
| 3 | Đọc danh sách card intervals timeline đã vật hóa; sắp xếp `TU_NGAY ASC, card_id ASC`, lấy trang. | Có card intervals? | Map timeline card DTO. | Nếu 0 trong khi result có timeline → MSG-E900; không trả timeline giả. | SQL-03 | Có thể hiển thị card bị hủy với isValid=false, nhưng không dùng nó để tính gap. |
| 4 | Đọc các gap periods nằm trong cửa sổ cá nhân; sắp xếp `gap_start ASC`. | Gap rows rỗng? | Rỗng là hợp lệ nếu tiêu chí không yêu cầu gap. | Nếu GAP_90_PLUS nhưng tổng gap <90 → lỗi nhất quán. | SQL-04 |  |
| 5 | Đối soát SUM(gap_days) với `total_gap_days` trên person_result. | Số tổng khớp? | Trả 200. | Không khớp → log ERROR và MSG-E900. | SQL-02/04 |  |
| 6 | Trả `cards`, `gaps`, metadata phân trang; mask PII theo role. | Có dữ liệu nhạy cảm? | Mask rồi trả. | Không log nội dung cards. | — |  |

## 4. Định nghĩa SQL / truy vấn dữ liệu

Mọi truy vấn dùng bind parameter, không ghép trực tiếp giá trị do client cung cấp vào SQL. Thời gian chờ truy vấn và statement timeout phải cấu hình; khi timeout, hủy truy vấn và trả lỗi có `traceId`.

### SQL-01 — Resolve snapshot

```sql
SELECT snapshot_id, status FROM hssv_batch_run
WHERE (:snapshotId IS NULL AND status='PUBLISHED') OR snapshot_id=:snapshotId
ORDER BY published_at DESC LIMIT 2;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId optional. |
| Index / partition | PK snapshot_id; published index |
| Số bản ghi kỳ vọng | 0..1 |
| Có dữ liệu | 1 row → continue. |
| Không có (NOT FOUND) | 0 row → 404; non-published → 409. |
| Nhiều hơn kỳ vọng | >1 → MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; 2s. |

### SQL-02 — Confirm person in published results and fetch totals

```sql
SELECT criteria, total_gap_days, gap_count, window_start, window_end
FROM hssv_person_result
WHERE snapshot_id=:snapshotId AND so_so_bhxh=:soSoBhxh
ORDER BY criteria LIMIT 3;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId + ID. |
| Index / partition | Unique composite keys |
| Số bản ghi kỳ vọng | 0..2 |
| Có dữ liệu | Result row allows timeline. |
| Không có (NOT FOUND) | 0 → 404. |
| Nhiều hơn kỳ vọng | >2 → MSG-E900. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; 2s. |

### SQL-03 — Read page of card timeline

```sql
SELECT card_id, ma_khoi, tu_ngay, den_ngay, trang_thai,
       dm_bhxh_id, ma_tinh, ten_tinh
FROM hssv_card_timeline
WHERE snapshot_id=:snapshotId AND so_so_bhxh=:soSoBhxh
ORDER BY tu_ngay ASC, card_id ASC
LIMIT :pageSize OFFSET :offset;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | pageSize <=100; offset from page. |
| Index / partition | Index `(snapshot_id, so_so_bhxh, tu_ngay, card_id)` |
| Số bản ghi kỳ vọng | 0..pageSize |
| Có dữ liệu | Rows map to cards[]. |
| Không có (NOT FOUND) | 0 rows allowed only if result has no stored timeline; otherwise MSG-E900. |
| Nhiều hơn kỳ vọng | More than pageSize is contract violation. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; 2s. |

### SQL-04 — Read gap periods

```sql
SELECT gap_seq, gap_start, gap_end, gap_days
FROM hssv_gap_period
WHERE snapshot_id=:snapshotId AND so_so_bhxh=:soSoBhxh
ORDER BY gap_start ASC
LIMIT 10000;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | snapshotId + ID. |
| Index / partition | Index `(snapshot_id, so_so_bhxh, gap_start)` |
| Số bản ghi kỳ vọng | 0..n (soft cap 10000) |
| Có dữ liệu | Rows map to gaps[]. |
| Không có (NOT FOUND) | 0 is valid for no gaps. |
| Nhiều hơn kỳ vọng | >10000 means data contract cap exceeded; return pagination/MSG-E900, do not silently truncate. |
| Lỗi DB / Timeout | DB error/timeout → MSG-E503/E504. |
| Khóa / Timeout | Read-only; 3s; paging required if cap is hit. |

## 5. Đặc tả kiểm tra đầu vào

| Check-ID | Hạng mục | Loại kiểm tra | Điều kiện lỗi | Mã thông báo | Thứ tự | Dừng ngay? | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CHK-01 | Authorization | RBAC | Không có quyền xem timeline | MSG-E401/E403 | 1 | Có | BD §6 |
| CHK-02 | ID/pagination | Format/range | ID không hợp lệ; page<1; pageSize ngoài 1..100 | MSG-E001/E002/E003 | 2 | Có | BD §8 |
| CHK-03 | Timeline integrity | Business consistency | GAP_90_PLUS mà totalGapDays<90 hoặc gap ra ngoài cửa sổ | MSG-E900 | 3 | Có | BD §4 |

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
| UT-001 | Bình thường | 2-6 | Có thẻ phủ nối tiếp và 1 gap 90 ngày | soSoBhxh eligible | GET timeline | Cards chronological; gap contains correct start/end/gapDays; total=90 |  |  |  |  |  |
| UT-002 | Bình thường | 3-5 | Người STOPPED không có gap model | soSoBhxh eligible stopped | GET timeline | Cards returned; gaps=[] or stored known gaps per period definition |  |  |  |  |  |
| UT-003 | Biên | 4 | Các khoảng có DEN/TU sát ngày | gap end 31/03, next start 01/04 | GET timeline | gapDays=0, không tạo khoảng gián đoạn |  |  |  |  |  |
| UT-004 | Bất thường | 5 | Mock totals differ between result and gap table | gap sum != total_gap_days | GET timeline | 500 MSG-E900, alert; không trả kết quả sai |  |  |  |  |  |
| UT-005 | Bất thường | 1-2 | Người không thuộc result snapshot | soSoBhxh unknown | GET timeline | 404 MSG-I404 |  |  |  |  |  |
| UT-006 | Hiệu năng | 3 | Người có nhiều thẻ | pageSize=100 | GET timeline | p95 <=1.5s mục tiêu, pagination stable |  |  |  |  |  |

## 8. QA / câu hỏi cần xử lý

| No | Ngày hỏi | Mục | Câu hỏi | Người trả lời | Trả lời/giả định đang dùng | Ngày trả lời | Trạng thái | Đã phản ánh? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-10-28 | 03/05 | Kiểm tra ngưỡng/ngày danh mục | Đã kế thừa quy tắc tuần 1–2; câu hỏi nghiệp vụ chưa phê duyệt được ghi ở Basic Design QA. | BA/PO | Giữ `>=90 ngày`; dùng danh mục gắn snapshot. | 2026-10-28 | Open | Chưa |
