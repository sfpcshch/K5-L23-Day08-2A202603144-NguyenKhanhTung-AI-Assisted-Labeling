# Số đo review (det)

- Ảnh tính: 24 (batch,gold); box máy: 502; đối tượng sau duyệt: 478; ngưỡng accepted: IoU >= 0.9
- Đã **loại 25 đối tượng nằm quanh các bẫy** khỏi bảng này: lỗi nhân tạo không phản ánh chất lượng máy.

## 1. Bốn trạng thái (AI error profile)
| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted (giữ nguyên) | 466 | 466/502 = 92.8% | 502 box máy |
| edited (sửa) | 3 | 3/502 = 0.6% | 502 box máy |
| deleted (xoá) | 33 | 33/502 = 6.6% | 502 box máy |
| added (vẽ thêm) | 9 | 9/478 = 1.9% | 478 đối tượng sau duyệt |

Trong số deleted + added, **7 trường hợp là đổi lớp** (máy gán sai lớp, box đúng chỗ): đó không phải máy bỏ sót. Số vật máy **sót thật** ≈ added − đổi lớp = 2.

### Theo lớp
| Lớp | accepted | edited | deleted | added |
|---|---|---|---|---|
| person | 262 | 0 | 7 | 0 |
| bicycle | 6 | 0 | 1 | 2 |
| motorbike | 138 | 0 | 2 | 1 |
| car | 35 | 0 | 3 | 0 |
| bus | 3 | 0 | 0 | 4 |
| truck | 2 | 0 | 4 | 1 |
| three_wheeler | 20 | 3 | 16 | 1 |

## 2. So với gold (nhãn gán tay từ đầu)
Ảnh gold: 4; đối tượng gold: 33 (mẫu số của recall)
| | TP | FP | FN | Precision | Recall |
|---|---|---|---|---|---|
| Máy (pre-label) | 21 | 48 | 12 | 30.4% (KTC 95%: 21–42%) | 63.6% (KTC 95%: 47–78%) |

*Lạc quan: ngưỡng được chọn trên chính các ảnh gold này (in-sample), và chỉ có 4 ảnh. Đọc khoảng tin cậy, đừng đọc con số đơn.*


(Ảnh gold không nằm trong nhãn sau duyệt: không tính residual / new error. Đó là chủ ý: duyệt lại ảnh bạn vừa tự gán tay sẽ đo trí nhớ, không đo thiên kiến.)

## 3. Ảnh bẫy
- Bắt được 6/12 = 50.0% (KTC 95%: 25–75%) (ngưỡng đạt: 80%). Chỉ 12 bẫy nên khoảng tin cậy rất rộng: đọc cùng danh sách bẫy lọt, đừng kết luận chỉ từ %.
  - wrong_class: 3/3
  - missing: 0/3
  - loose: 1/3
  - extra: 2/3
  - lọt: img_003.jpg (missing), img_009.jpg (loose), img_029.jpg (missing), img_013.jpg (loose), img_010.jpg (missing), img_024.jpg (extra)

## Gợi ý đọc số
- `deleted` cao → ngưỡng thấp hoặc ánh xạ nhãn sai; `added` cao → máy bỏ sót (xem lớp nào);
- `accepted` thấp (< 50%) → sửa mỗi box lâu hơn vẽ mới: đổi model/ngưỡng hoặc vẽ tay;
- bắt bẫy < 80% → duyệt lại lô, siết checklist (quét ảnh TRƯỚC khi nhìn nhãn máy).
