# Bộ tài liệu thiết kế hệ thống BHYT HSSV — 200M dòng

Bộ tài liệu này được xây dựng từ yêu cầu tuần hiện tại, các mẫu Basic Design/Detail Design được cung cấp và repository `Mnn2kkk6/bhyt_hssv`. Đây là **tài liệu thiết kế**, không chứa code triển khai.

## Cấu trúc bộ tài liệu

- `BasicDesign_HSSV_200M.md` — bản Markdown dùng để review, quản lý version và AI đọc.
- `BasicDesign_HSSV_200M.xlsx` — bản Excel theo mẫu Basic Design, gồm tổng quan, chức năng, API, batch, bảng dữ liệu, API contract, mã lỗi, NFR và QA.
- `DetailDesign/` — 8 bản Detail Design Markdown và 8 bản Excel tương ứng:
  - `DD-BAT-HSSV-001_XuLySnapshot_200M` — xử lý snapshot 200M dòng.
  - `DD-API-HSSV-001_DanhSachNguoi` — `GET /api/v1/hssv/persons`.
  - `DD-API-HSSV-002_ChiTietNguoi` — `GET /api/v1/hssv/persons/{soSoBhxh}`.
  - `DD-API-HSSV-003_Timeline` — `GET /api/v1/hssv/persons/{soSoBhxh}/timeline`.
  - `DD-API-HSSV-004_BaoCaoTinh` — `GET /api/v1/hssv/reports/provinces`.
  - `DD-API-HSSV-005_PhanBoGianDoan` — `GET /api/v1/hssv/reports/distribution`.
  - `DD-API-HSSV-006_Export` — `POST /api/v1/hssv/reports/export`.
  - `DD-API-HSSV-007_ExportJobStatus` — API phụ trợ `GET /api/v1/hssv/reports/export/{jobId}`.
- `MaTranTruyVet.md` — ánh xạ yêu cầu → chức năng/API/batch → DD → test.

## Cách đọc / review

1. Review Basic Design trước để chốt phạm vi, quy tắc nghiệp vụ, kiến trúc và API contract.
2. Xem từng DD theo đúng API-ID/BAT-ID; bước xử lý được đánh số; SQL-ID có trường hợp có dữ liệu, không có dữ liệu, nhiều hơn kỳ vọng, lỗi DB và timeout.
3. Đóng các mục `Open` trong Basic Design QA trước khi triển khai. Bản hiện tại đặt trạng thái `Draft`, chưa điền người review/phê duyệt.
4. Khi thiết kế được duyệt, mỗi UT-ID có thể được ánh xạ sang một unit-test method; test integration/load/security cần được bổ sung khi bước triển khai.

## Nguồn nghiệp vụ được kế thừa từ repository

- `README.md`: phạm vi Ý 1 dừng tham gia và Ý 2 gián đoạn.
- `solve.py`: thẻ hợp lệ, tháng gốc 10/2024, tháng kiểm tra dừng mặc định 08/2026, quy tắc chọn thẻ gốc.
- `solve_y2.py`: mốc cuối 07/2026, cửa sổ cá nhân, gộp khoảng, cách tính gap, tie-break, transaction idempotent.
- `test_y2.py`: ca biên 90/89 ngày, thẻ hủy, nối tiếp/chồng lấn/lồng nhau, cửa sổ cá nhân, đổi khối.
- `y2_ddl_queries.sql`: bảng kết quả, thống kê tỉnh và bucket phân bố hiện có.
- `Tuan 2/`: phân tích nghiệp vụ, luồng dữ liệu, thuật toán và test case.

## Quyết định kiến trúc là đề xuất thiết kế

Repository hiện tại là pipeline Python/Pandas/Parquet/SQLite trên dữ liệu giả lập, cấu hình 20 triệu người. Để thiết kế cho 200M dòng, bộ tài liệu đề xuất object storage/Parquet cho raw và curated, Spark cho xử lý phân tán, Airflow để điều phối, PostgreSQL read model cho API, Redis cache và worker cho export. Đây là kiến trúc mục tiêu; chưa có benchmark hay triển khai xác nhận.

## Hai điểm phải xác nhận trước khi code

1. Tuần 2 đã chốt **tổng gián đoạn `>=90 ngày`**, nhưng đề bài mới dùng cụm “>3 tháng”. Tài liệu tạm kế thừa `>=90` theo code/test hiện có; cần BA/PO xác nhận.
2. API tỉnh/TP dùng phiên bản danh mục gắn với snapshot. Repo có danh mục 63 tỉnh/TP; cần Data Owner xác nhận nếu snapshot mục tiêu dùng danh mục lịch sử hay danh mục khác.

Các mục tiêu p95, timeout, retention, maxRows và TTL export đều là giá trị thiết kế/placeholder cần benchmark hoặc phê duyệt; không phải kết quả kiểm thử thực tế.
