# Thiết kế cơ bản (Basic Design)
## Hệ thống xử lý dữ liệu BHYT HSSV quy mô 200 triệu dòng

| Thuộc tính | Giá trị |
|---|---|
| Mã tài liệu | HSSV-BD-001 |
| Phiên bản | 1.0 |
| Trạng thái | Draft – chờ review/phê duyệt |
| Ngày lập | 2026-10-28 |
| Người lập / review / phê duyệt | Chưa điền |
| Repo tham chiếu | https://github.com/Mnn2kkk6/bhyt_hssv |
| Đầu vào nghiệp vụ | Bài tập tuần 1–2; README, `solve.py`, `solve_y2.py`, `y2_ddl_queries.sql`, `test_y2.py` |
| Phạm vi tài liệu | Thiết kế; chưa bao gồm hiện thực code |

## 1. Mục đích và phạm vi

Thiết kế hệ thống có khả năng xử lý tập dữ liệu thẻ BHYT khoảng 200.000.000 dòng để xác định hai nhóm kết quả HSSV: (1) đã dừng tham gia BHYT; (2) vẫn tham gia ở mốc hiện tại nhưng có tổng thời gian gián đoạn cộng dồn đạt ngưỡng nghiệp vụ. Kết quả được lưu ở dạng phục vụ tra cứu, thống kê theo tỉnh/TP, thống kê phân bố ngày gián đoạn và xuất Excel/CSV.

### Trong phạm vi
- Nhận và kiểm tra snapshot dữ liệu thẻ `ST_INTHE_DTL` cùng danh mục `DM_BHXH`.
- Làm sạch/chuẩn hóa dữ liệu, gắn phiên bản snapshot và xử lý dữ liệu lớn theo lô.
- Xác định kết quả dừng tham gia và gián đoạn theo đúng quy tắc tuần 1–2.
- Cung cấp sáu API được yêu cầu và một API phụ trợ theo dõi job xuất báo cáo bất đồng bộ.
- Báo cáo theo 63 tỉnh/TP của danh mục dữ liệu đầu vào; phân bố tổng ngày gián đoạn.
- Kiểm soát truy cập, audit, quan sát vận hành, tính đúng đắn và khả năng chạy lại.

### Ngoài phạm vi
- Ghi/cập nhật dữ liệu BHYT nguồn hoặc quyết định quyền lợi bảo hiểm.
- Tự động kết luận pháp lý thay cho cán bộ nghiệp vụ.
- Giao diện web hoàn chỉnh, tích hợp với hệ thống BHXH thật, xác thực định danh quốc gia.
- Hiện thực code, triển khai production và kiểm thử hiệu năng trên hạ tầng thật trong tuần thiết kế.

## 2. Thuật ngữ và quy tắc nghiệp vụ

| Thuật ngữ | Định nghĩa |
|---|---|
| Thẻ hợp lệ | Bản ghi có `TRANG_THAI = 1`; trạng thái khác 1 không được dùng để xác định thời gian được bảo phủ. |
| Tháng gốc | Mặc định 2024-10; tìm người có thẻ HS/SV giao tháng này. |
| Tháng kiểm tra dừng | Mặc định 2026-08; chỉ cần có thẻ hợp lệ bất kỳ khối nào giao tháng là người đó chưa được xếp vào nhóm dừng. |
| Tháng cuối gián đoạn | Mặc định 2026-07; người phải có thẻ hợp lệ bất kỳ khối nào giao tháng này. |
| Giao một tháng | `TU_NGAY <= ngày cuối tháng` và `DEN_NGAY >= ngày đầu tháng`; hai đầu mút của thẻ đều có hiệu lực. |
| Cửa sổ cá nhân gián đoạn | Từ ngày phủ thẻ đầu tiên trong tháng gốc tới ngày phủ thẻ cuối cùng trong tháng cuối. Không mặc định các mốc này giống nhau cho mọi người. |
| Khoảng gián đoạn | Số ngày không được bất kỳ thẻ hợp lệ nào phủ giữa hai khoảng đã gộp: `TU_NGAY_tiếp - DEN_NGAY_trước - 1`. |
| Ngưỡng gián đoạn | Theo `solve_y2.py` và `test_y2.py`: tổng ngày trống cộng dồn `>= 90`. Đúng 90 ngày đạt; 89 ngày không đạt. |
| Tỉnh báo cáo | Tỉnh/TP được suy ra từ cơ quan BHXH của thẻ gốc; trong bảng timeline vẫn lưu cơ quan gắn với từng thẻ. |

**Giả định cần phê duyệt:** cụm “gián đoạn > 3 tháng” trong yêu cầu tuần này được xử lý theo quy tắc đã dùng trong bài tập tuần 2 (`>= 90 ngày`). Nếu nghiệp vụ yêu cầu nghiêm ngặt `> 90 ngày`, cần đổi rule, report và test biên 90/91 ngày trước khi code. Danh mục yêu cầu hiện tại ghi 63 tỉnh/TP; thiết kế dùng đúng snapshot danh mục trong repo và không hard-code tên tỉnh ngoài danh mục nguồn.

## 3. Kiến trúc đề xuất

```text
Nguồn snapshot Parquet + DM_BHXH
             |
             v
  Landing / Object Storage (raw, bất biến)
             |
             v
 Airflow: validate -> normalize -> repartition -> classify -> quality checks
             |
             v
 Spark batch (200M dòng; partition theo khóa người tham gia)
             |
       +-----+------------------------+
       |                              |
       v                              v
Curated Parquet                 Kết quả batch đã kiểm tra
(the/timeline)                  (dừng / gián đoạn / gap)
       |                              |
       +--------------+---------------+
                      v
         Read Model PostgreSQL
  person_result | card_interval | gap_period
  province_report | gap_distribution | export_job
                      |
        +-------------+-------------+
        |                           |
        v                           v
 FastAPI behind API Gateway     Worker xuất file
        |                           |
        v                           v
 RBAC + audit + cache Redis    Object Storage CSV/XLSX
        |
        v
  Client tra cứu / hệ thống báo cáo
```

### Thành phần và trách nhiệm

| Thành phần | Trách nhiệm |
|---|---|
| Landing/Object Storage | Lưu snapshot nguồn bất biến, file lỗi, curated Parquet và kết quả xuất; không nạp 200M dòng vào API database. |
| Apache Airflow (đề xuất) | Điều phối DAG, retry theo bước, log trạng thái batch, kiểm tra phụ thuộc và publish một snapshot kết quả hoàn chỉnh. |
| Apache Spark (đề xuất) | Đọc/phân vùng dữ liệu lớn, sắp xếp theo người và thời gian, gộp khoảng thẻ, tính gap và kết quả theo lô. |
| PostgreSQL | Read model đã lọc để API đọc nhanh; partition theo snapshot/kỳ, index theo mã BHXH, tỉnh, khối và tiêu chí. |
| Redis | Cache báo cáo tổng hợp và danh mục ít thay đổi; không làm nguồn dữ liệu chính. |
| FastAPI | Validate request, xác thực/phân quyền, gọi read model, trả response JSON chuẩn. |
| Export Worker + Object Storage | Tạo CSV/XLSX bất đồng bộ, giới hạn hàng/cột, lưu link tải có thời hạn. |
| Observability | Log JSON có `traceId`/`snapshotId`, metrics, alert thời gian batch, bản ghi lỗi, thời gian phản hồi và job xuất file. |

Đây là kiến trúc mục tiêu để xử lý quy mô 200M, không phải mô tả công nghệ đang có trong repository. Repo hiện tại dùng Python/Pandas/Parquet/SQLite cho bộ dữ liệu giả lập 20M người; khi triển khai 200M dòng cần benchmark và chốt kích thước cluster, partition, index cùng giới hạn SLA.

## 4. Quy trình xử lý batch

1. **Đăng ký snapshot:** ghi `snapshotId`, mốc gốc, mốc kiểm tra dừng, tháng cuối gián đoạn, ngưỡng ngày, phiên bản danh mục và thời điểm nhận file.
2. **Kiểm tra đầu vào:** schema bắt buộc, kiểu dữ liệu, null khóa, định dạng ngày, miền `TRANG_THAI`, checksum và số dòng. File/dòng không hợp lệ vào quarantine; vượt ngưỡng lỗi cấu hình thì dừng batch.
3. **Chuẩn hóa và ghi curated:** chuẩn hóa mã, kiểu ngày, trạng thái và định danh; giữ bản raw; không xóa vật lý dữ liệu nguồn. Khử trùng lặp chỉ theo khóa nghiệp vụ đã được phê duyệt; không tự suy diễn khi bản ghi xung đột.
4. **Repartition theo người:** nhóm toàn bộ thẻ của `SO_SO_BHXH` vào cùng một nhóm xử lý logic. Không dựa vào giả định hiện tại của bộ fake data rằng lịch sử một người luôn nằm trong cùng file part.
5. **Lọc thẻ hợp lệ:** `TRANG_THAI = 1`; tính giao khoảng tháng theo hai đầu mút bao gồm.
6. **Tính nhóm dừng:** G = người có thẻ HS/SV giao tháng 10/2024; K = người có bất kỳ thẻ hợp lệ nào giao tháng 08/2026; kết quả = G trừ K.
7. **Tính nhóm gián đoạn:** G = người có thẻ HS/SV giao tháng 10/2024; K = người có bất kỳ thẻ hợp lệ nào giao tháng 07/2026; ứng viên = G giao K. Tính cửa sổ cá nhân, cắt các thẻ vào cửa sổ, gộp khoảng chồng lấn/nối tiếp bằng mốc kết thúc lớn nhất đến trước đó, cộng số ngày trống; giữ tổng `>= 90`.
8. **Chọn thẻ đại diện:** giữ đúng tie-break tuần 1–2: thẻ gốc ưu tiên `DEN_NGAY` muộn nhất, `TU_NGAY` muộn nhất, ưu tiên SV rồi `ID` lớn nhất; thẻ đại diện tháng cuối ưu tiên ngày phủ trong tháng muộn nhất, `TU_NGAY` muộn nhất, `ID` lớn nhất.
9. **Ghi read model:** ghi kết quả, timeline rút gọn và khoảng gián đoạn; tổng hợp báo cáo theo danh mục tỉnh/TP (bao gồm tỉnh có số lượng 0) và các nhóm ngày gián đoạn.
10. **Kiểm tra và publish:** xác nhận khóa duy nhất, thống kê bằng tổng chi tiết, số tỉnh trả về, tổng gap, số hàng reject; chỉ chuyển `snapshotId` sang `PUBLISHED` sau khi toàn bộ kiểm tra đạt. Khi lỗi, giữ snapshot trước đó đang được phục vụ.
11. **Retry/idempotency:** mỗi run có khóa `(snapshotId, baseMonth, stopCheckMonth, gapEndMonth, threshold, ruleVersion)`; retry không tạo bản ghi trùng; lần chạy thất bại không publish kết quả một phần.

## 5. Danh sách chức năng

| ID | Chức năng | Loại | Mô tả |
|---|---|---|---|
| F-HSSV-001 | Xử lý snapshot và phân loại | Batch | Xử lý dữ liệu lớn, phát hiện dừng tham gia/gián đoạn, kiểm tra chất lượng và publish snapshot. |
| F-HSSV-002 | Tra cứu danh sách | API | Lọc theo tỉnh, khối và tiêu chí; phân trang và sắp xếp ổn định. |
| F-HSSV-003 | Tra cứu chi tiết người | API | Trả kết quả phân loại và thông tin thẻ đại diện đã được phân quyền. |
| F-HSSV-004 | Xem timeline thẻ và khoảng trống | API | Trả chuỗi thẻ theo thời gian, trạng thái hợp lệ và các khoảng gián đoạn. |
| F-HSSV-005 | Thống kê theo tỉnh/TP | API | Trả 63 dòng theo danh mục snapshot, gồm cả tỉnh/TP không có kết quả. |
| F-HSSV-006 | Thống kê phân bố gián đoạn | API | Thống kê số người theo bucket ngày gián đoạn và khối HS/SV. |
| F-HSSV-007 | Xuất báo cáo | API + Worker | Tạo CSV/XLSX theo bộ lọc; xử lý bất đồng bộ với file tải có thời hạn. |
| F-HSSV-008 | Theo dõi job xuất | API phụ trợ | Truy vấn trạng thái job xuất và link tải khi hoàn tất; cần thiết để hoàn chỉnh luồng export bất đồng bộ. |

## 6. Danh sách API

Các route dưới đây dùng snapshot `PUBLISHED` mới nhất nếu không truyền `snapshotId`. `snapshotId` giúp tái lập truy vấn lịch sử.

| ID API | Method & endpoint | Mục đích | SLA mục tiêu thiết kế |
|---|---|---|---|
| API-HSSV-001 | `GET /api/v1/hssv/persons` | Tra cứu danh sách, lọc theo tỉnh/khối/tiêu chí, phân trang. | p95 ≤ 1.000 ms, `pageSize` tối đa 100. |
| API-HSSV-002 | `GET /api/v1/hssv/persons/{soSoBhxh}` | Xem chi tiết người trong tập kết quả. | p95 ≤ 800 ms. |
| API-HSSV-003 | `GET /api/v1/hssv/persons/{soSoBhxh}/timeline` | Timeline thẻ và các khoảng gián đoạn. | p95 ≤ 1.500 ms cho tối đa 100 item/page. |
| API-HSSV-004 | `GET /api/v1/hssv/reports/provinces` | Thống kê theo 63 tỉnh/TP trong danh mục snapshot. | p95 ≤ 1.000 ms khi cache/summary sẵn sàng. |
| API-HSSV-005 | `GET /api/v1/hssv/reports/distribution` | Phân bố tổng ngày gián đoạn. | p95 ≤ 1.000 ms khi summary sẵn sàng. |
| API-HSSV-006 | `POST /api/v1/hssv/reports/export` | Tạo job CSV/XLSX theo bộ lọc. | Trả `202 Accepted` trong ≤ 1.000 ms; thời gian file phụ thuộc dung lượng. |
| API-HSSV-007 | `GET /api/v1/hssv/reports/export/{jobId}` | API phụ trợ: xem trạng thái job và nhận link tải file. | p95 ≤ 500 ms. |

### Quy ước request chung
- Xác thực: OAuth2/JWT; mọi endpoint cần role tra cứu phù hợp, export cần thêm quyền `HSSV_REPORT_EXPORT`.
- Request có `traceId` do gateway/API sinh nếu caller không cung cấp; ghi audit truy cập.
- Mọi response lỗi có `code`, `message`, `traceId`; không trả stacktrace/SQL/PII đầy đủ.
- `criteria`: `STOPPED` hoặc `GAP_90_PLUS`. `block`: `HS` hoặc `SV`.
- Phân trang danh sách: `page` bắt đầu từ 1, `pageSize` mặc định 20, tối đa 100; sắp xếp có tie-break `SO_SO_BHXH` để kết quả ổn định.
- `snapshotId` mặc định là snapshot mới nhất trạng thái `PUBLISHED`; không truy vấn snapshot đang xử lý.
- Kết quả cá nhân và timeline chỉ cung cấp trường cần thiết; CMND/CCCD, số thẻ và địa chỉ phải được che/mask theo role và chính sách dữ liệu.

## 7. Mô hình dữ liệu

| ID | Đối tượng logic | Lưu trữ | Mục đích / khóa chính |
|---|---|---|---|
| TBL-HSSV-001 | `raw_card_snapshot` | Object Storage Parquet | Dữ liệu thẻ nguồn bất biến theo snapshot; 200M dòng, phân vùng theo snapshot và bucket/hash người sau bước chuẩn hóa. |
| TBL-HSSV-002 | `curated_card_interval` | Object Storage Parquet | Thẻ chuẩn hóa dùng cho tính toán; giữ `ID`, `SO_SO_BHXH`, `MA_KHOI`, `TU_NGAY`, `DEN_NGAY`, `TRANG_THAI`, `DM_BHXH_ID`. |
| TBL-HSSV-003 | `hssv_person_result` | PostgreSQL | Một người / snapshot / tiêu chí; khóa `(snapshot_id, criteria, so_so_bhxh)`. |
| TBL-HSSV-004 | `hssv_card_timeline` | PostgreSQL | Timeline thẻ của những người thuộc tập kết quả; khóa `(snapshot_id, so_so_bhxh, card_id)`. |
| TBL-HSSV-005 | `hssv_gap_period` | PostgreSQL | Một khoảng trống / người / snapshot; khóa `(snapshot_id, so_so_bhxh, gap_seq)`. |
| TBL-HSSV-006 | `hssv_province_report` | PostgreSQL | Một dòng/tỉnh với bốn counters: dừng HS/SV và gián đoạn HS/SV; chứa tỉnh có số lượng 0. |
| TBL-HSSV-007 | `hssv_gap_distribution` | PostgreSQL | Tổng hợp số người theo bucket ngày, tiêu chí snapshot và khối. |
| TBL-HSSV-008 | `hssv_export_job` | PostgreSQL | Trạng thái job xuất, bộ lọc đã chuẩn hóa, file key, thời điểm hết hạn và lỗi đã che. |
| TBL-HSSV-009 | `hssv_batch_run` | PostgreSQL | Theo dõi run, tham số, rule version, số dòng vào/ra/reject, checkpoint và trạng thái. |
| TBL-HSSV-010 | `dm_bhxh_versioned` | PostgreSQL / Parquet | Danh mục cơ quan BHXH, ánh xạ tỉnh/TP theo phiên bản dữ liệu. |

### Chỉ mục và phân vùng đề xuất
- `hssv_person_result`: partition theo snapshot/tháng kết quả; B-tree `(snapshot_id, criteria, ma_tinh, ma_khoi, so_so_bhxh)`; index lookup `(snapshot_id, so_so_bhxh)`.
- `hssv_card_timeline`: `(snapshot_id, so_so_bhxh, tu_ngay, den_ngay)`; partition cùng phiên bản snapshot.
- `hssv_gap_period`: `(snapshot_id, so_so_bhxh, gap_start)`; `hssv_gap_distribution` và `hssv_province_report` đọc theo khóa summary.
- Tránh truy vấn API trực tiếp 200M dòng raw; API chỉ truy cập bảng serving đã được publish.
- Chỉ mục/partition thực tế phải được benchmark với thống kê cardinality và truy vấn representative trước khi chốt.

## 8. Định nghĩa request/response API

### API-HSSV-001 — Danh sách người
Request query: `provinceCode?`, `block?`, `criteria?`, `snapshotId?`, `page=1`, `pageSize=20`, `sort=soSoBhxh`, `direction=asc|desc`.

Response 200:
```json
{
  "snapshotId": "hssv-2026-08-r001",
  "criteria": "GAP_90_PLUS",
  "page": 1,
  "pageSize": 20,
  "totalItems": 120000,
  "totalPages": 6000,
  "items": [
    {
      "soSoBhxh": "masked-or-authorized-value",
      "fullName": "Nguyen Van A",
      "block": "SV",
      "provinceCode": "01",
      "provinceName": "Theo danh muc snapshot",
      "resultType": "GAP_90_PLUS",
      "totalGapDays": 90
    }
  ],
  "traceId": "..."
}
```

### API-HSSV-002 — Chi tiết người
Response 200 gồm `snapshotId`, `soSoBhxh`, thông tin định danh tối thiểu theo quyền, `results[]` (criteria, block, province, representative card dates, totalGapDays, gapCount), `generatedAt`. Không tìm thấy trong snapshot hoặc không có quyền xem thì trả 404/403 theo chính sách chống dò tìm định danh.

### API-HSSV-003 — Timeline
Request query: `snapshotId?`, `page=1`, `pageSize=100` (max 100). Response gồm `cards[]` (cardId, block, startDate, endDate, status, agency/province code) và `gaps[]` (startDate, endDate, gapDays, gapSequence), `windowStart`, `windowEnd`, `totalGapDays`. Thẻ bị hủy vẫn có thể hiển thị để giải thích lịch sử nếu role cho phép, nhưng không được tính là thời gian bảo phủ. Timeline được sắp xếp tăng dần theo ngày bắt đầu rồi cardId.

### API-HSSV-004 — Báo cáo tỉnh/TP
Request query: `snapshotId?`, `criteria?`, `block?`. Response `items[]` có đúng 63 dòng theo danh mục snapshot; mỗi dòng có `provinceCode`, `provinceName`, `stoppedStudentCount`, `stoppedUniversityStudentCount`, `stoppedTotalCount`, `gap90PlusStudentCount`, `gap90PlusUniversityStudentCount`, `gap90PlusTotalCount`. Nếu query có `criteria`, client chỉ dùng nhóm metrics của tiêu chí đó; response vẫn giữ 63 dòng và zero-count provinces.

### API-HSSV-005 — Phân bố gián đoạn
Request query: `snapshotId?`, `provinceCode?`, `block?`. Bucket theo quy ước hiện tại: `090-119`, `120-179`, `180-269`, `270_PLUS` ngày. Response gồm từng bucket, số người, tỷ trọng trong tổng tập đủ điều kiện, và phân tách HS/SV. Không dùng `SO_THANG_QUY_DOI` làm tiêu chí lọc vì trường này chỉ hiển thị (`ngày / 30`).

### API-HSSV-006 — Export
Request JSON: `format=CSV|XLSX`, `criteria?`, `provinceCode?`, `block?`, `snapshotId?`, `columns[]?`, `maxRows?`. API validate allowlist cột, giới hạn số dòng theo cấu hình và tạo job. Response 202 gồm `jobId`, `status=QUEUED`, `statusUrl`, `createdAt`; không trả file nhị phân ngay trong request.

### API-HSSV-007 — Export job status (phụ trợ)
`GET /api/v1/hssv/reports/export/{jobId}` trả `QUEUED|RUNNING|SUCCEEDED|FAILED|EXPIRED`, `progressPercent?`, `downloadUrl?`, `expiresAt?`, `errorCode?`. URL tải là URL có chữ ký và thời hạn; không đoán tên file từ jobId.

## 9. Mã thông báo

| Mã | HTTP | Nội dung chính | Áp dụng |
|---|---:|---|---|
| MSG-E001 | 400 | Thiếu hoặc rỗng tham số bắt buộc. | ID người, format, query |
| MSG-E002 | 400 | Tham số sai định dạng hoặc ngoài miền cho phép. | Ngày, mã tỉnh, block, criteria |
| MSG-E003 | 400 | Tham số phân trang không hợp lệ. | page/pageSize |
| MSG-E004 | 400 | Cấu trúc bộ lọc/cột xuất không được hỗ trợ. | export |
| MSG-E401 | 401 | Chưa xác thực hoặc token không hợp lệ. | toàn bộ API |
| MSG-E403 | 403 | Không có quyền thực hiện thao tác. | detail/timeline/export |
| MSG-I404 | 404 | Không tìm thấy kết quả trong snapshot được chọn. | detail/timeline/export job |
| MSG-E409 | 409 | Snapshot chưa publish hoặc trạng thái hiện tại không cho phép thao tác. | truy vấn snapshot |
| MSG-E413 | 413 | Yêu cầu xuất vượt giới hạn cấu hình. | export |
| MSG-E429 | 429 | Quá giới hạn tần suất; thử lại sau. | toàn bộ API |
| MSG-E500 | 500 | Có lỗi nội bộ; cung cấp traceId khi liên hệ vận hành. | toàn bộ API |
| MSG-E503 | 503 | Dịch vụ dữ liệu tạm thời không sẵn sàng/timeout. | DB/object store |
| MSG-E504 | 504 | Quá thời gian chờ theo SLA. | truy vấn/worker |
| MSG-E900 | 500 | Snapshot không nhất quán; kết quả chưa được publish. | kiểm tra batch |

## 10. Yêu cầu phi chức năng

| Hạng mục | Mục tiêu thiết kế | Cách kiểm chứng |
|---|---|---|
| Khả năng mở rộng | Xử lý 200M dòng không cần nạp toàn bộ vào RAM một tiến trình; xử lý chia partition và shuffle theo mã người. | Benchmark với tỷ lệ 25M/100M/200M dòng; theo dõi peak RAM, spill, duration. |
| Hiệu năng API | p95 mục tiêu: list ≤1s, detail ≤0,8s, timeline ≤1,5s, summary ≤1s; các giá trị là mục tiêu, cần load test xác nhận. | k6/JMeter với dữ liệu đại diện; cache warm/cold. |
| Tính đúng đắn | Test biên 90/89; trạng thái hủy; khoảng chồng lấn/nối tiếp/lồng nhau; người đổi khối/tỉnh; reconcile thống kê. | Unit, integration, reconciliation tự động. |
| Tính nhất quán | Snapshot chỉ publish nguyên tử sau khi quality checks pass; truy vấn lịch sử chỉ rõ snapshotId. | Test lỗi giữa batch và kiểm tra snapshot trước vẫn đọc được. |
| Bảo mật | OAuth2/JWT, RBAC, che PII trong API/log, audit, link export hết hạn. | Security tests, kiểm tra log và quyền. |
| Khả dụng | Mục tiêu 99,5% trong giờ làm việc; cần chốt với nghiệp vụ/vận hành. | Monitoring, alert và diễn tập khôi phục. |
| Quan sát | Structured JSON logs có traceId/snapshotId/jobId; metrics batch, API latency, DB pool, cache, export duration. | Dashboard và alert runbook. |
| Lưu trữ | Raw snapshot bất biến; TTL cho export; quy định retention của kết quả và audit cần được phê duyệt. | Kiểm tra lifecycle và truy cập. |

## 11. Tiêu chí nghiệm thu thiết kế

- Mọi route và trường request/response có định nghĩa, kiểu, bắt buộc, miền giá trị và mã lỗi.
- Có sơ đồ luồng xử lý 200M dòng, chiến lược partition/repartition, retry, checkpoint và publish snapshot.
- Hai thuật toán nghiệp vụ khớp quy tắc tuần 1–2; test 90 ngày đạt và 89 ngày không đạt.
- Mỗi SQL-ID có số dòng kỳ vọng và cách xử lý không dữ liệu, dữ liệu bất thường, lỗi DB, timeout.
- Mỗi nhánh logic quan trọng có ít nhất một test case.
- Tỉnh/TP không có dữ liệu vẫn xuất hiện trong báo cáo theo snapshot danh mục đã chọn.
- API export có giới hạn, xử lý bất đồng bộ và cơ chế trạng thái/tải file.
- Mã lỗi nhất quán; có xác thực, phân quyền, audit, ẩn PII.
- Reviewer và người phê duyệt không phải người lập; các câu hỏi phạm vi phải được đóng trước khi code.

## 12. QA / điểm cần chốt

| ID | Câu hỏi | Quyết định đang dùng | Trạng thái |
|---|---|---|---|
| QA-HSSV-001 | “>3 tháng” là `>90` hay `>=90` ngày? | Kế thừa code/test tuần 2: `>=90`; T01 90 ngày đạt, T02 89 không đạt. | Open – cần xác nhận nghiệp vụ |
| QA-HSSV-002 | Mốc dữ liệu/API mặc định có cố định hay chọn theo snapshot? | Dùng snapshot `PUBLISHED` mới nhất; mặc định các mốc 10/2024, 08/2026 (dừng), 07/2026 (gián đoạn). | Proposed |
| QA-HSSV-003 | Từ “63 tỉnh/TP” cần theo danh mục lịch sử trong nguồn hay danh mục hiện hành? | API trả đúng phiên bản danh mục gắn snapshot; bộ dữ liệu repo hiện có danh mục 63 tỉnh/TP. | Open – xác nhận version danh mục |
| QA-HSSV-004 | Người không thuộc tập kết quả có được xem chi tiết/timeline? | Mặc định không; 404 nếu không có trong read model của snapshot. | Proposed |
| QA-HSSV-005 | Giới hạn xuất tối đa và thời gian lưu file? | Chưa chốt; cấu hình vận hành, export async; URL có chữ ký và TTL. | Open |
| QA-HSSV-006 | SLA batch và phần cứng cluster? | Chưa chốt; phải benchmark dữ liệu đại diện trước khi cam kết. | Open |

## 13. Tài liệu chi tiết đi kèm

- `DD-BAT-HSSV-001_XuLySnapshot_200M.md` — xử lý dữ liệu theo lô.
- `DD-API-HSSV-001_DanhSachNguoi.md`
- `DD-API-HSSV-002_ChiTietNguoi.md`
- `DD-API-HSSV-003_Timeline.md`
- `DD-API-HSSV-004_BaoCaoTinh.md`
- `DD-API-HSSV-005_PhanBoGianDoan.md`
- `DD-API-HSSV-006_Export.md`
- `DD-API-HSSV-007_ExportJobStatus.md` — API phụ trợ cho export bất đồng bộ.

