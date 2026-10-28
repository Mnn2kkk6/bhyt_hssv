# Thiết kế chi tiết — Batch xử lý snapshot 200M dòng

| Mục | Nội dung |
|---|---|
| Tên dự án | Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200M |
| Hệ thống | Hệ thống tra cứu kết quả BHYT HSSV |
| Tài liệu | Thiết kế chi tiết (Detail Design) |
| Mã tài liệu | DD-BAT-HSSV-001 |
| Phiên bản / trạng thái | 1.0 / Draft – chờ review |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Tài liệu cơ bản | `HSSV-BD-001` |
| Chức năng | F-HSSV-001 |
| Nguồn | Repo `https://github.com/Mnn2kkk6/bhyt_hssv`; quy tắc tuần 1–2; template Detail Design đã cung cấp |

## 1. Tổng quan module

| Mục | Nội dung |
|---|---|
| ID module | DD-BAT-HSSV-001 |
| Hiện thực cho | F-HSSV-001; Basic Design `HSSV-BD-001` |
| Mức chi tiết | B – API nghiệp vụ / A – batch quy mô lớn |
| Tóm tắt xử lý | Xác thực snapshot đầu vào → chuẩn hóa → repartition theo người → tính hai tiêu chí → tạo timeline/gap → tổng hợp → đối soát → publish nguyên tử. |
| Lớp/package đề xuất | `HssvController` → `HssvQueryService` / `HssvExportService` → `HssvReadRepository` / `ExportJobRepository` |
| Chữ ký phương thức | `BatchRunResponse runSnapshot(SnapshotRunRequest request)` |
| Được gọi bởi | API Gateway / batch orchestrator |
| Gọi tới | PostgreSQL read model, Redis cache nếu phù hợp, Object Storage nếu cần |
| Giao dịch | Đọc dùng read-only transaction; thao tác tạo job dùng transaction ngắn; không giữ transaction trong lúc tạo file |
| Xử lý đồng thời | Snapshot chỉ đọc ở trạng thái `PUBLISHED`; keyset/sắp xếp ổn định; export job có trạng thái và khóa idempotency |

## 2. Định nghĩa vào/ra

| Hướng | No | Tên tham số / thuộc tính | Kiểu | Bắt buộc | Mô tả / miền giá trị | Giá trị khi không có | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| IN | 1 | sourceSnapshot | Object Storage prefix | Có | Path raw immutable chứa snapshot thẻ; schema phải tương thích schema version đã phê duyệt. | — | BD §4 |
| IN | 2 | baseMonth | YYYY-MM | Có | Mặc định 2024-10; tháng gốc để xác định HS/SV. | 2024-10 | BD §4 |
| IN | 3 | stopCheckMonth | YYYY-MM | Có | Mặc định 2026-08; tháng kiểm tra dừng tham gia. | 2026-08 | BD §4 |
| IN | 4 | gapEndMonth | YYYY-MM | Có | Mặc định 2026-07; tháng cuối gián đoạn. | 2026-07 | BD §4 |
| IN | 5 | gapThresholdDays | Integer | Có | Số nguyên dương; hiện áp dụng >=90 theo tuần 2. | 90 | QA-HSSV-001 |
| OUT | 1 | snapshotId | String | Có | Định danh phiên bản kết quả. | — | BD §6 |
| OUT | 2 | status | Enum | Có | QUEUED/RUNNING/VALIDATING/PUBLISHED/FAILED. | — | TBL-HSSV-009 |
| OUT | 3 | inputRows/outputRows/rejectRows | Long | Có | Số lượng được xử lý và bị loại. | 0 | BD §7 |

## 3. Xử lý chi tiết

| Bước | Nội dung xử lý | Điều kiện | Khi đúng | Khi sai | SQL / Check / MSG-ID | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Nhận yêu cầu chạy batch; tạo `snapshotId` và `runId`, lưu tham số cùng version nghiệp vụ/danh mục. | Request có đủ tham số và chưa có run đang xử lý cùng idempotency key. | Ghi trạng thái QUEUED rồi bắt đầu run. | Sai tham số → MSG-E002; duplicate key → trả run đã có. | CHK-01, SQL-01, MSG-E002 | Không log dữ liệu cá nhân. |
| 2 | Kiểm tra input manifest, checksum, danh sách part, schema và metadata. | Thiếu file hoặc schema không tương thích? | Nếu hợp lệ sang bước 3. | Dừng run, FAILED; lưu lý do; không tác động snapshot đang publish. | CHK-02, SQL-01, MSG-E900 |  |
| 3 | Quét từng file/partition, kiểm tra kiểu ngày, mã trạng thái, khóa người; ghi dòng lỗi vào quarantine. | Dòng lỗi <= ngưỡng cấu hình và schema hợp lệ? | Chuẩn hóa dòng đúng; đếm reject; tiếp tục. | Nếu reject tỷ lệ vượt ngưỡng hoặc lỗi khóa nghiêm trọng → dừng và cảnh báo. | CHK-03, MSG-E900 | Ngưỡng cụ thể cấu hình vận hành, chưa chốt. |
| 4 | Chuyển raw thành curated Parquet; chuẩn hóa ngày, kiểu mã và status; bảo toàn raw snapshot. | Đầu ra curated tạo thành công? | Ghi manifest curated và số dòng. | Lỗi storage/serialization → retry theo partition; quá số lần → FAILED. | SQL-01, MSG-E503 |  |
| 5 | Repartition/shuffle theo `SO_SO_BHXH`, bảo đảm mọi thẻ của một người được tính trong cùng logical group. | Partition manifest đầy đủ? | Tiếp tục phân loại. | Nếu có partition thiếu/khóa null → dừng run, không tính thiếu người. | CHK-03, MSG-E900 | Không dựa giả định mỗi người nằm cùng physical input file. |
| 6 | Lọc thẻ hợp lệ `TRANG_THAI=1`; tính tập G có HS/SV giao baseMonth và tập K có thẻ bất kỳ giao stopCheckMonth. | G/K đã dựng đủ theo snapshot? | Tính kết quả dừng = G trừ K. | Nếu ngày sai hoặc không xác định được tháng → FAILED. | CHK-04 | Hai đầu mút khoảng thẻ là inclusive. |
| 7 | Tính tập ứng viên gián đoạn: G có HS/SV giao baseMonth giao với K có thẻ bất kỳ giao gapEndMonth. | Ứng viên rỗng? | Rỗng vẫn là kết quả hợp lệ; sang bước tổng hợp zero. | Có ứng viên → sang bước 8. | CHK-04 |  |
| 8 | Với từng ứng viên, tính cửa sổ cá nhân, cắt các thẻ vào cửa sổ, gộp chồng lấn/nối tiếp và tính gap = TU tiếp theo - DEN trước - 1. | Có các khoảng phủ sau khi cắt? | Cộng các gap dương, đếm số khoảng; giữ người có tổng >= gapThresholdDays. | Không có khoảng phủ hợp lệ → ghi lỗi chất lượng, không tự suy diễn. | CHK-05, MSG-E900 | Phải dùng cumulative maximum của DEN để xử lý interval lồng nhau. |
| 9 | Chọn thẻ gốc và thẻ đại diện tháng cuối theo tie-break đã duyệt. | Có nhiều thẻ ứng viên tương đương? | Dùng DEN desc, TU desc, ưu tiên SV, ID desc cho thẻ gốc; ngày phủ desc, TU desc, ID desc cho thẻ cuối. | Nếu tie-break không tạo duy nhất → báo dữ liệu không nhất quán. | CHK-06 |  |
| 10 | Quy đổi `DM_BHXH_ID` sang tỉnh/TP theo version danh mục của snapshot. | Không ánh xạ được cơ quan? | Gắn mã `XX`/`Không xác định` và đếm exception theo chính sách. | Nếu tỷ lệ unknown vượt ngưỡng, dừng publish và cảnh báo. | SQL-02, MSG-E900 | Không tự gán tỉnh thay thế. |
| 11 | Ghi bảng kết quả theo snapshot vào staging tables; ghi timeline và gap periods. | Write thành công? | Sang đối soát. | Lỗi DB → rollback transaction/partition write; retry idempotent. | SQL-02, MSG-E503 | Không ghi đè snapshot đang publish. |
| 12 | Đối soát số dòng, khóa trùng, 90/89 test fixture, summary so với detail, tỉnh danh mục và bucket distribution. | Tất cả quality checks pass? | Sang bước 13. | Nếu fail, trạng thái FAILED; không publish. | SQL-03, CHK-07, MSG-E900 |  |
| 13 | Publish nguyên tử bằng cách cập nhật trạng thái snapshot và active pointer trong transaction ngắn. | Snapshot đủ dữ liệu và chưa bị thay thế bởi run mới hơn? | Đổi trạng thái PUBLISHED; invalidate cache báo cáo. | Không đủ điều kiện → conflict; giữ active pointer cũ. | SQL-03, MSG-E900 |  |
| 14 | Ghi metrics, run duration, partition throughput, reject ratio và trạng thái cuối. | Batch kết thúc? | Ghi SUCCEEDED/PUBLISHED cùng thống kê. | Lỗi ghi metric không được làm mất trạng thái batch đã publish; ghi WARN. | SQL-03 |  |
| E | Xử lý exception không bắt được ở bước bất kỳ. | Lỗi không thể retry? | Ghi FAILED và alert có runId/snapshotId. | Retry chỉ với bước idempotent; không publish dữ liệu dở dang. | MSG-E503/MSG-E900 | Raw snapshot và active result hiện hành không bị xóa. |

## 4. Định nghĩa SQL / truy vấn dữ liệu

Mọi truy vấn dùng bind parameter, không ghép trực tiếp giá trị do client cung cấp vào SQL. Thời gian chờ truy vấn và statement timeout phải cấu hình; khi timeout, hủy truy vấn và trả lỗi có `traceId`.

### SQL-01 — Tạo/đọc batch run và manifest

```sql
SELECT run_id, snapshot_id, status, input_manifest_uri, rule_version
FROM hssv_batch_run
WHERE idempotency_key = :idempotencyKey
ORDER BY created_at DESC
LIMIT 2;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | `:idempotencyKey` được tạo từ snapshot + mốc + threshold + ruleVersion. |
| Index / partition | IDX_BATCH_IDEMPOTENCY; index snapshot/status |
| Số bản ghi kỳ vọng | 0..2 |
| Có dữ liệu | 0: tạo run mới; 1: tiếp tục hoặc trả run tương ứng. |
| Không có (NOT FOUND) | Không có run cũ là tình huống bình thường. |
| Nhiều hơn kỳ vọng | >1 active run cùng key → kiểm tra corruption/lock, không khởi chạy duplicate. |
| Lỗi DB / Timeout | DB lỗi → MSG-E503; timeout → dừng và retry có giới hạn. |
| Khóa / Timeout | Transaction ngắn; timeout 3s. |

### SQL-02 — Ghi staging result summary

```sql
INSERT INTO hssv_person_result
(snapshot_id, criteria, so_so_bhxh, ma_khoi, ma_tinh, total_gap_days, result_payload)
VALUES (:snapshotId, :criteria, :soSoBhxh, :block, :provinceCode, :gapDays, :payload)
ON CONFLICT (snapshot_id, criteria, so_so_bhxh)
DO UPDATE SET ma_khoi=EXCLUDED.ma_khoi,
              ma_tinh=EXCLUDED.ma_tinh,
              total_gap_days=EXCLUDED.total_gap_days,
              result_payload=EXCLUDED.result_payload;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | Dữ liệu từ partition đã qua validate. |
| Index / partition | PK `(snapshot_id, criteria, so_so_bhxh)` |
| Số bản ghi kỳ vọng | Một hàng mỗi người/criteria/snapshot |
| Có dữ liệu | 1 write thành công; zero rows từ partition là hợp lệ. |
| Không có (NOT FOUND) | Không có output ở partition thì không gọi insert. |
| Nhiều hơn kỳ vọng | Conflict xử lý idempotent; mismatch cùng key cần ghi metric. |
| Lỗi DB / Timeout | DB error → rollback partition chunk, retry; timeout → retry giới hạn. |
| Khóa / Timeout | Batch write theo chunk; không giữ transaction toàn bộ 200M dòng. |

### SQL-03 — Đối soát và publish snapshot

```sql
SELECT criteria, COUNT(*) AS row_count,
       COUNT(DISTINCT so_so_bhxh) AS distinct_people
FROM hssv_person_result
WHERE snapshot_id = :snapshotId
GROUP BY criteria;
```

| Thuộc tính | Giá trị |
|---|---|
| Tham số bind | `:snapshotId` là snapshot đang ở trạng thái VALIDATING. |
| Index / partition | PK/filter snapshot + criteria; summary được lưu sau đối soát. |
| Số bản ghi kỳ vọng | 0..2 nhóm criteria |
| Có dữ liệu | Rows và distinct people phải khớp; kết quả summary được đóng dấu PASS. |
| Không có (NOT FOUND) | Không có kết quả có thể hợp lệ nếu source không có ứng viên; vẫn phải tạo summary 0. |
| Nhiều hơn kỳ vọng | Duplicate khóa hoặc count lệch → FAILED/MSG-E900, không publish. |
| Lỗi DB / Timeout | DB error/timeout → không đổi active pointer. |
| Khóa / Timeout | Publish pointer cùng transaction ngắn; timeout 5s. |

## 5. Đặc tả kiểm tra đầu vào

| Check-ID | Hạng mục | Loại kiểm tra | Điều kiện lỗi | Mã thông báo | Thứ tự | Dừng ngay? | Tham chiếu BD |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CHK-01 | Tham số run | Required/format | Thiếu snapshot hoặc tháng sai `YYYY-MM` | MSG-E001/E002 | 1 | Có | BD §4 |
| CHK-02 | Manifest/schema | Schema contract | Thiếu part/checksum hoặc cột bắt buộc | MSG-E900 | 2 | Có | BD §4 |
| CHK-03 | Khóa và ngày | Data quality | `SO_SO_BHXH` null, ngày không parse được, khoảng ngày đảo; tỷ lệ lỗi vượt config | MSG-E900 | 3 | Theo severity | BD §4 |
| CHK-04 | Điều kiện giao tháng | Business rule | Thẻ không hợp lệ bị dùng làm coverage hoặc tháng đối chiếu không đủ | MSG-E900 | 4 | Có | BD §4 |
| CHK-05 | Tính khoảng trống | Boundary | Không cắt theo cửa sổ cá nhân, bỏ qua interval lồng nhau hoặc đếm thiếu 1 ngày | MSG-E900 | 5 | Có | UT-005 |
| CHK-06 | Tie-break đại diện | Determinism | Cùng đầu vào chạy lại chọn thẻ đại diện khác khi không có thay đổi nguồn | MSG-E900 | 6 | Có | UT-006 |
| CHK-07 | Reconciliation | Cross-table | Chi tiết/summary lệch, snapshot có duplicate key hoặc thiếu tỉnh 0-result | MSG-E900 | 7 | Có | BD §11 |

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
| UT-B001 | Bình thường | 1-4 | Manifest schema hợp lệ, nguồn có nhiều partition | 200M rows; config mặc định | Chạy DAG đến VALIDATING | Tạo staging snapshot, ghi số dòng; raw bất biến |  |  |  |  |  |
| UT-B002 | Bất thường | 2 | Thiếu part/checksum | manifest thiếu một file | Chạy validate | FAILED; không publish snapshot mới; snapshot cũ vẫn active |  |  |  |  |  |
| UT-B003 | Bất thường | 3 | Tỷ lệ dòng lỗi vượt ngưỡng | ngày parse lỗi / khóa null | Chạy data quality | Quarantine + dừng run; MSG-E900 |  |  |  |  |  |
| UT-B004 | Biên | 6-8 | Ca T01 từ test_y2 | gap = 90 ngày | Chạy rule gap | Người được giữ trong GAP_90_PLUS |  |  |  |  |  |
| UT-B005 | Biên | 6-8 | Ca T02 từ test_y2 | gap = 89 ngày | Chạy rule gap | Người bị loại khỏi GAP_90_PLUS |  |  |  |  |  |
| UT-B006 | Bất thường | 8 | Interval lồng nhau; thẻ hủy phủ vào gap | Dữ liệu T05/T06 | Chạy merge intervals | Cummax đúng; thẻ hủy không lấp khoảng trống |  |  |  |  |  |
| UT-B007 | Bình thường | 10-13 | Snapshot hợp lệ; một tỉnh có 0 kết quả | publish run | Chạy reconcile/publish | Có đủ tỉnh theo danh mục; pointer chuyển nguyên tử sau PASS |  |  |  |  |  |
| UT-B008 | Đồng thời | 11-13 | Worker retry cùng idempotency key | Chạy song song 2 lần | Khởi chạy lại DAG | Không nhân đôi kết quả; một active publish |  |  |  |  |  |

## 8. QA / câu hỏi cần xử lý

| No | Ngày hỏi | Mục | Câu hỏi | Người trả lời | Trả lời/giả định đang dùng | Ngày trả lời | Trạng thái | Đã phản ánh? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 2026-10-28 | 03/05 | Ngưỡng gián đoạn là >=90 hay >90? | BA/PO | Đang kế thừa >=90 ngày từ tuần 2; cần PO xác nhận. | 2026-10-28 | Open | Có trong BD QA-HSSV-001 |
| 2 | 2026-10-28 | 03/05 | Ngưỡng reject và phần cứng cluster? | Tech Lead/Ops | Phải chốt bằng benchmark; không giả định SLA batch trước đo thử. | 2026-10-28 | Open | Có trong BD QA-HSSV-006 |
