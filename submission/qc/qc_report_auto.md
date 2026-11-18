# Báo cáo QC tự động (nháp)

> **Gợi ý** do script tính; bạn xác nhận. Script chỉ đếm theo phiếu kiểm bằng mắt người.

## Mẫu
- 7 ảnh (2 ca khó + 5 ngẫu nhiên), 120 đối tượng.
- taxonomy v1.0

## Số đo
| Chỉ số | Giá trị | Mẫu số |
|---|---|---|
| Lỗi **theo đối tượng** | 2/120 = 1.7% | 120 đối tượng trong mẫu |
| **Ảnh có lỗi** (dùng cho ngưỡng) | 2/7 = 28.6% | 7 ảnh trong mẫu |
| Máy: accepted / edited / deleted | 466/502 = 92.8% / 3/502 = 0.6% / 33/502 = 6.6% | 502 box máy |
| Máy: added | 9/478 = 1.9% | 478 đối tượng sau duyệt |
| Bắt lỗi bẫy | 6/12 = 50.0% | 12 bẫy |

## Top 3 loại lỗi
| Loại lỗi | Số lỗi | Ảnh ví dụ |
|---|---|---|
| thua | 2 | img_010.jpg, img_022.jpg |

## Giới hạn thống kê
- 2 ảnh lỗi trong 7 ảnh mẫu: cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là **64%**. Ngưỡng 2% không thể được **chứng minh** bằng mẫu nhỏ như vậy: hãy nói rõ giới hạn này trong báo cáo.

## Gợi ý quyết định
**REWORK**

- ảnh có lỗi 28.6% > 2%
- bắt bẫy 50% < 80% -> lô duyệt chưa đáng tin

Cây quyết định: Hold (guideline chưa rõ / dữ liệu hỏng) -> Rework (> ngưỡng, lỗi lặp, bắt bẫy / nhất quán thấp) -> Release.
