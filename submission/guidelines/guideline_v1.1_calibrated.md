# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603144 - Nguyễn Khánh Tùng

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (và ghi rõ phạm vi áp dụng)
> cùng quy tắc mới rút ra từ lỗi của máy / box `can_review`. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp `submission/qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Xe đạp điện / xe máy điện: `bicycle` hay `motorbike`? (Mục 6.1) | Có bàn đạp nhìn thấy được tính là `bicycle`; không có bàn đạp (xe máy điện / tay ga điện) tính là `motorbike`. | `img_015.jpg`, `img_030.jpg` | `CAL-01` |
| 2 | Khối xe máy đỗ dày đặc và vật lẻ quá nhỏ `< 20 px` (Mục 6.2) | Cụm ≥ 5 xe đỗ sát nhau hoặc cao `< 20 px` gom bằng 1 `ignore_region`; xe lẻ cao `< 20 px` xoá hẳn không gán. | `img_018.jpg`, `img_034.jpg` | — |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_010.jpg`, `img_002.jpg`, `img_013.jpg` | Chụp góc sau của xích lô có cấu tạo đuôi xe đạp gắn ghế chở khách phía trước; hoặc tuk-tuk giống ô tô/xe tải nhỏ | YOLO26m bắt nhầm phần đuôi xích lô thành `#115 bicycle` (`conf 0.50`) nằm lọt trong `#114 three_wheeler`; hoặc gán tuk-tuk thành `car`/`truck` | Không tách riêng bộ phận của xe ba bánh; xoá hẳn box `bicycle` con `#115` và gán toàn xe là `three_wheeler` (ưu tiên YOLOE khi merge, sửa tay nếu sót) | **Mục 2 & 4.1 (`CAL-02`):** Xe ba bánh (xích lô, ba gác, tuk-tuk) luôn là **1 box `three_wheeler` duy nhất** ôm cả thân, phần đuôi xe đạp/xe máy gắn liền và mái che; cấm gán thêm box `bicycle`/`motorbike` cho bộ phận cấu thành. |
| `img_022.jpg` | Ảnh chợ đêm tối/nhòe ở rìa trái khung hình (`#253`), mắt người không nhìn rõ có phải người hay vật tĩnh | YOLO26m dự đoán `person` điểm thấp (`conf 0.35`) khiến người duyệt phải đánh dấu `can_review` | Xoá hẳn nhãn thay vì giữ box `can_review` mơ hồ | **Mục 4.3 (`CAL-03`):** Nếu đối tượng ở xa/tối/mờ mà sau khi zoom tối đa mắt người vẫn không thể xác nhận chắc chắn thuộc lớp nào thì **không gán** (xoá bỏ pre-label). |
| `img_003.jpg`, `img_009.jpg`, `img_013.jpg`, `img_029.jpg` | Bỏ sót 3/3 bẫy `missing` và 2/3 bẫy `loose` khi duyệt nhanh (`~1.0 phút/ảnh`) | Máy tạo bẫy xoá mất box hoặc làm lệch khung rõ rệt | Lỗi thuộc quy trình duyệt (thiên kiến tự động hóa), không sửa định nghĩa guideline | **Quy trình Review (`CAL-04`):** Không đổi câu chữ định nghĩa lớp v1.1; bắt buộc Rework với bước ẩn nhãn máy 15 giây đầu và rà 4 cạnh khung. |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt quy tắc phân loại xe điện theo bàn đạp; cấm gán tách bộ phận đuôi xe đạp của xích lô/tuk-tuk; quy tắc xoá hẳn box quá tối/mờ không xác nhận được; chuẩn hóa xử lý cụm xe nhỏ `< 20 px`. | Tổng hợp từ kết quả tour model, box `can_review` (`img_010.jpg`, `img_022.jpg`), lỗi lặp của máy (`img_002.jpg`, `img_013.jpg`) và kiểm tra QC: `CAL-01`, `CAL-02`, `CAL-03`, `CAL-04`. |
