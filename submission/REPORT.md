# Báo cáo Day 08 — 2A202603144 - Nguyễn Khánh Tùng

> Báo cáo tổng hợp toàn bộ quy trình thực nghiệm Day 08 (Part 1–6). Mọi tỉ lệ trong bài đều đi kèm **mẫu số** và trích trực tiếp từ đầu ra script trong `submission/qc/`, `submission/time_log.csv` và `submission/model_log.csv`.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / bus / truck | `YOLO26m` | Mặc định `0.30` (qua `--conf 0.3`) | Trên 4 ảnh gold, `person` (1 vật), `bicycle` (4 vật), `bus` (1 vật), `truck` (0 vật) đều `< 5` vật nên không đủ mẫu chọn ngưỡng riêng; `motorbike` (15 vật) không có ngưỡng `<= 0.5` nào đạt recall ≥ 90% (tối đa 80%) nên giữ ngưỡng mặc định `0.30` (`r02`, `r03`). |
| car | `YOLO26m` | `0.50` | Đạt recall `100%` (`6/6` vật gold) tại ngưỡng `0.50` với precision `46%`, lọc bớt box xe nhiễu ở ngưỡng thấp (`r03`, `threshold_sweep.md`). |
| three_wheeler | `YOLOE-26s` (`'tuk-tuk'`, `'cyclo'`, `'cargo tricycle'`) | `0.15` | Closed-set COCO không có lớp `three_wheeler`; ngưỡng `0.15` đạt recall `100%` (`6/6` vật gold) với precision `67%`, đồng thời đặt `yoloe_all` đứng trước `y26_all` khi merge để thắng các box `car`/`truck` trùng (`r04`). |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên):
  `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "car=0.5,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM đã chọn** (`SAM 2.1 Base` — `sam2.1_b.pt`): Tách biên mask chi tiết hơn hẳn `mobile_sam.pt` trên xe ba bánh nhiều thanh khung hở và xe đạp có giỏ; tốc độ chạy GPU RTX 3050 Ti (4 GB VRAM) đạt ~`1.25 giây/ảnh` ở `r05`, dung lượng gọn (155 MB) và dùng trực tiếp qua `ultralytics` mà không cần xin quyền truy cập như `facebook/sam3`.
- **Lỗi không sửa được bằng đổi ngưỡng:** `YOLO26m` thuộc mô hình closed-set 80 lớp COCO (không có khái niệm `three_wheeler`), nên ở `img_002.jpg` tuk-tuk vẫn bị gọi là `car` với điểm rất cao `0.89`, còn ở `img_010.jpg` phần đuôi xe đạp gắn sau xích lô vẫn bị bắt riêng thành `bicycle` (`0.50`). Dù tăng hay giảm `--conf`, `YOLO26m` không thể tự sinh nhãn `three_wheeler`; phải giải quyết bằng mô hình open-vocabulary (`YOLOE-26s`) ưu tiên cao hơn khi gộp + hậu kiểm bằng mắt.
- **Box điểm cao mà sai / điểm thấp mà đúng:** Ở `img_002.jpg`, tuk-tuk bị `YOLO26m` gán `car` với điểm `0.89` (cao nhưng sai lớp); ở `img_010.jpg`, phần đuôi xe đạp của xích lô `#115` được gán `bicycle` điểm `0.50` (thừa vì nằm bên trong xe ba bánh). Ngược lại, ở `img_010.jpg`, `YOLOE-26s` bắt đúng hai chiếc xích lô xếp hàng phía sau (`#112` và `#113` `three_wheeler`) với điểm thấp chỉ `0.16` và `0.26` (nếu giữ ngưỡng mặc định `0.30` hoặc `0.50` sẽ bỏ sót).
- **Giới hạn chọn ngưỡng trên gold:** Việc quét và chọn ngưỡng được thực hiện trực tiếp trên **cùng 4 ảnh gold** (`in-sample`), và tổng số mẫu quá nhỏ (nhiều lớp `< 5` vật), nên chỉ số precision/recall của máy trên gold mang tính **lạc quan** và ngưỡng chọn chỉ là ước lượng thô.

## 2. AI error profile (từ `qc/review_stats.md`)

*(Tính trên 24 ảnh batch sau khi **loại 25 đối tượng nằm quanh vùng 12 bẫy** để lỗi nhân tạo không làm méo chất lượng thực của model.)*

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 466 | 466/502 = 92.8% | 502 box máy |
| edited | 3 | 3/502 = 0.6% | 502 box máy |
| deleted | 33 | 33/502 = 6.6% | 502 box máy |
| added | 9 | 9/478 = 1.9% | 478 đối tượng sau duyệt |

- **Lớp máy tệ nhất:** `three_wheeler` là lớp tệ nhất về độ nhiễu/dương tính giả với `deleted = 16/39` box máy (tổng 39 box `three_wheeler` sau loại bẫy: `20 accepted, 3 edited, 16 deleted`), chủ yếu do ngưỡng `0.15` của `YOLOE-26s` sinh nhiều box thừa/trùng ở ảnh xe ba bánh (`img_002.jpg`, `img_010.jpg`, `img_017.jpg`). Tiếp theo là `truck` (`deleted = 4/6` box máy) và `bus` (`added = 4/7` đối tượng sau duyệt, máy hay bỏ sót hoặc nhầm xe buýt ở xa). Trong 33 box `deleted` và 9 box `added`, có **7 ca đổi lớp** nên số vật máy sót thật là `9 - 7 = 2`.
- **So với gold (nhãn gán tay từ đầu):** Precision của máy `21/69 = 30.4%` (KTC 95%: 21–42%, mẫu số `69` box dự đoán `TP + FP` trên 4 ảnh gold); Recall của máy `21/33 = 63.6%` (KTC 95%: 47–78%, mẫu số `33` đối tượng thực tế trên 4 ảnh gold).

## 3. Thiên kiến tự động hóa và ảnh bẫy

- **Bắt bẫy:** `6/12 = 50.0%` (KTC 95%: 25–75%, dưới ngưỡng đạt `80%`). Theo từng loại bẫy:
  - `wrong_class`: `3/3` (bắt tốt nhờ màu/tên lớp đập vào mắt khi nhìn lướt).
  - `extra`: `2/3` (lọt 1 ca ở `img_024.jpg`, nơi đường phố quá dày đặc xe máy/người).
  - `loose`: `1/3` (lọt 2 ca ở `img_009.jpg` và `img_013.jpg`).
  - `missing`: `0/3` (lọt cả 3 ca ở `img_003.jpg`, `img_010.jpg`, `img_029.jpg`).
  *(Ghi chú phân tích chi tiết ở `img_010.jpg`: Thực tế khi duyệt mắt, tôi đã nhận ra xích lô bị mất box và tự vẽ thêm `#114 three_wheeler [42.7, 349.7, 376.6, 841.4]`, nhưng vẽ theo phần thân dưới và bánh xe mà thiếu phần mái che đỏ phía trên so với box chuẩn `[47.8, 197.0, 379.9, 659.4]`, dẫn đến `IoU = 0.472 < 0.50` và bị script chấm là lọt bẫy `missing`.)*
- **Vì sao lọt & liên hệ thiên kiến tự động hóa:** Tỉ lệ `accepted` rất cao (`466/502 = 92.8%`) đi kèm tỉ lệ bắt bẫy thấp (`6/12 = 50.0%`, đặc biệt `0/3 missing` và `1/3 loose`) là dấu hiệu điển hình của **thiên kiến tự động hóa (automation bias)** và **hiệu ứng neo nhãn**: do tâm lý chủ quan làm nhanh (`24.1 phút / 24 ảnh ≈ 1.00 phút/ảnh`), tôi đã không thực hiện nghiêm túc Bước 0 (ẩn hết nhãn máy để tự quét ảnh trước) mà nhìn trực tiếp vào các box pre-label sẵn có. Khi mắt bám theo các box máy đã vẽ sẵn, các box bị thiếu (`missing` ở góc/nền hoặc vùng đông như `img_003`, `img_029`) và các box bị lệch biên vài chục pixel (`loose` ở `img_009`, `img_013`) rất dễ bị bỏ qua.

## 4. Máy có đáng dùng không?

- **Thời gian (từ `time_log.csv`):** Gán tay 4 ảnh gold hết `20.9 phút` (`20.9/4 = 5.23 phút/ảnh`), trong khi duyệt 24 ảnh batch có pre-label hết `24.1 phút` (`24.1/24 = 1.00 phút/ảnh`, nhanh hơn ~`81%`). **Thiên lệch cần lưu ý:** 4 ảnh gold được gán trước nên người gán đã quen thuộc bố cục giao thông và quy tắc trước khi vào batch; đồng thời tốc độ `1.00 phút/ảnh` ở batch cũng phản ánh việc duyệt quá nhanh dẫn đến lọt 6/12 bẫy (nếu duyệt chuẩn theo checklist thì tốc độ thực tế nên ở mức ~2.5–3.0 phút/ảnh, vẫn nhanh hơn gán tay ~45–50%).
- **Kết luận:** **Có đáng dùng**, đặc biệt cho các lớp phổ biến mà detector đạt hiệu năng cao (`person` `262 accepted`, `motorbike` `138 accepted`, `car` `35 accepted`) và dùng `YOLOE-26s` để định vị nhanh `three_wheeler`, **với điều kiện** bắt buộc kết hợp checklist chống thiên kiến. Riêng các lớp máy hay sai/sót như `three_wheeler` (box con) và `bus` vẫn cần người duyệt kiểm tra kỹ.
- **Ba quy tắc giảm thiên kiến (đo kiểm được):**
  1. **Ẩn nhãn máy bắt buộc 15 giây đầu:** Ở **100% ảnh (24/24)**, nhấn phím `H` ẩn toàn bộ nhãn máy tối thiểu **15 giây đầu tiên** để tự đếm số người/xe ở từng góc phần tư trước khi hiện lại box pre-label (giảm lọt `missing` từ 3/3 xuống 0).
  2. **Kiểm soát tốc độ duyệt sàn và nghỉ giữa phiên:** Giữ thời gian duyệt trung bình **≥ 2.5 phút/ảnh** (tối thiểu `60 phút / 24 ảnh`, đối chiếu qua `time_log.csv` và CVAT telemetry) và nghỉ 5 phút sau mỗi 10 ảnh để tránh chủ quan lướt nhanh.
  3. **Kiểm tra 4 cạnh box lớn trước khi chuyển ảnh:** Với **100% box có kích thước > 40×40 px**, phóng to kiểm tra đủ 4 cạnh (bao gồm cả mái che xe ba bánh theo quy tắc 4.1) đảm bảo sai số biên ≤ 2 px trước khi sang ảnh kế tiếp (khắc phục lọt `loose` ở `img_009`, `img_013` và lỗi IoU `0.472` ở `img_010`).

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_013.jpg` | `SAM 2.1 Base` (`sam2.1_b.pt`) | 1 box + 7 click (3 dương, 4 âm) | Không gồm người lái (đã tách sạch đầu, áo xanh và chân tài xế nhờ 4 click âm, chỉ lấy thân xe và mái tuk-tuk). | Mask đạt `192982 px` (`27.6%` ảnh, 2 polygon); tách sạch người ngồi ăn phía sau qua khung hở nhưng hơi rỗ nhẹ ở phần lốp trước và vết chói sáng trên mái/lưng ghế. | `submission/masks/tuktuk_img013.jpg` |
| `img_002.jpg` | `SAM 2.1 Base` (`sam2.1_b.pt`) | 1 box + 5 click (2 dương, 3 âm) | Không gồm tài xế và không dính người đi xe máy phía trước bên phải. | Mask đạt `42284 px` (`6.1%` ảnh, 2 polygon); tách rất gọn khung xanh và bánh sau tuk-tuk, bỏ qua nền người đi bộ xuyên qua khung hở, chỉ sót phần mép bánh trước bị người đi xe máy che khuất. | `submission/masks/tuktuk_occluded_img002.jpg` |
| `img_015.jpg` | `SAM 2.1 Base` (`sam2.1_b.pt`) | 1 box + 5 click (3 dương, 2 âm) | Không gồm người đạp xe; **có gồm giỏ hàng + túi vải gắn cố định phía trước** (đúng quy tắc hàng gắn cố định). | Mask đạt `35991 px` (`6.1%` ảnh, 2 polygon); tách chuẩn quần jean và áo khoác người lái ra khỏi khung xe đạp, tuy nhiên không khoét được các khe nan hoa nhỏ ở bánh xe. | `submission/masks/bicycle_basket_img015.jpg` |

- **Đối chiếu guideline mục 4.2:** Quy tắc 4.2.1 (bám biên ≤ 3 px), 4.2.2 (bỏ qua lỗ nhỏ < 5% diện tích như khe nan hoa ở `img_015.jpg`, nhưng bỏ phần trống lớn nhìn xuyên khung xích lô/tuk-tuk ở `img_002.jpg`, `img_013.jpg`) và 4.2.4 (hai vật chạm nhau thì vùng chạm thuộc vật phía trước như người lái xe máy che bánh trước tuk-tuk ở `img_002.jpg`) giúp chuẩn hóa việc đặt click âm (`--neg`). Tuy nhiên mục 4.2 cần nhấn mạnh thêm hướng dẫn xử lý **túi/giỏ hàng nhô cao phía trước xe đạp** (`img_015.jpg`) đồng bộ với box để tránh gán thừa hoặc thiếu phần giỏ gắn cố định.
- **Thử nghiệm Interactor SAM trong CVAT (tùy chọn):** Không bật serverless Nuclio trên Docker local nên không dùng Interactor trực tiếp trong CVAT; toàn bộ 3 mask chạy qua `scripts/sam2_points.py` trên GPU (ưu điểm: tái lập được 100% tọa độ click và lưu trực tiếp ảnh minh chứng; nhược điểm: phải tự tra tọa độ pixel thay vì xem mask tức thì sau từng click như giao diện CVAT).

## 6. Calibration (Part 6)

- **Số quyết định:** `4` quyết định (`CAL-01` đến `CAL-04` trong `submission/qc/calibration_decisions.csv`). Quyết định quan trọng nhất là **`CAL-02`** (`img_010.jpg`, `img_002.jpg`, `img_013.jpg`): xác định rõ xe ba bánh (xích lô/tuk-tuk) chỉ được gán **1 box `three_wheeler` duy nhất** ôm toàn bộ thân xe và mái che, tuyệt đối xóa bỏ box `bicycle`/`motorbike` con mà detector tách nhầm từ phần đuôi xe đạp hoặc đầu xe máy gắn liền (như lỗi `#115 bicycle` trên đuôi xích lô ở `img_010.jpg`).
- **Quy tắc mới đã vào `guideline_v1.1_calibrated.md`:**
  1. `CAL-01`: Phân định xe đạp điện và xe máy điện theo tiêu chí **có bàn đạp nhìn thấy được** → `bicycle`, không có bàn đạp → `motorbike` (`img_015.jpg`, `img_030.jpg`).
  2. `CAL-02`: Sửa tuk-tuk bị gọi là `car`/`truck` về `three_wheeler` và xóa mọi box bộ phận xe đạp/xe máy con bên trong xe ba bánh (`img_010.jpg`, `img_002.jpg`).
  3. `CAL-03`: Xóa hẳn các box máy `conf` thấp ở vùng quá tối/nhòe mà mắt người zoom tối đa vẫn không xác nhận được, không giữ lại kèm `can_review` (`#253 person` ở `img_022.jpg`).
  4. Chốt bổ sung mục 6.2 cho cụm xe máy dày đặc / `< 20 px` (`img_018.jpg`, `img_034.jpg`).
- **Phân biệt nguyên nhân gốc ("guideline mơ hồ" vs "tôi / máy sai"):** `CAL-01` và `CAL-03` là **guideline v1.0 còn mơ hồ** (chưa quy định tiêu chí bàn đạp xe điện và cách xử lý khi zoom hết cỡ vẫn không thấy rõ vật), nên bắt buộc cập nhật vào `guideline_v1.1_calibrated.md`. Ngược lại, việc YOLO26 gọi tuk-tuk là `car` là **lỗi hạn chế của máy**, còn việc lọt 6/12 bẫy (`CAL-04`) và giữ box `#115` / `#253` do chưa quyết định xóa là **lỗi quy trình duyệt chủ quan của người gán**, không đổi định nghĩa lớp của guideline.

## 7. QC và quyết định (Part 6)

- **Mẫu QC:** `7/24` ảnh tập batch (`2` ảnh có `can_review` là `img_010.jpg`, `img_022.jpg` + `5` ảnh ngẫu nhiên `img_003.jpg`, `img_006.jpg`, `img_014.jpg`, `img_017.jpg`, `img_034.jpg`, `seed = 42`), tổng `120` đối tượng. **Hạn chế:** Người thực hiện kiểm tra QC chính là người đã duyệt lô nhãn trước đó (dễ bỏ sót lỗi giống lúc duyệt), thiếu kiểm tra chéo độc lập.
- **Tỉ lệ lỗi (từ `qc_report_auto.md`):**
  - Lỗi theo **đối tượng**: `2/120 = 1.7%` (mẫu số `120` đối tượng trong 7 ảnh mẫu).
  - Lỗi theo **ảnh**: `2/7 = 28.6%` (mẫu số `7` ảnh mẫu, vượt xa ngưỡng Release `≤ 2%`).
- **Top lỗi ghi nhận:** `thua` (`2` lỗi):
  - `img_010.jpg` (vật `#115`): box `bicycle` (`conf 0.50`, đã tick `can_review`) bắt nhầm phần đuôi xe đạp của chiếc xích lô `#114 three_wheeler` — cần xóa hẳn theo `CAL-02`.
  - `img_022.jpg` (vật `#253`): box `person` (`conf 0.35`, đã tick `can_review`) ở rìa trái ảnh chợ đêm quá mờ không khẳng định được bằng mắt người — cần xóa hẳn theo `CAL-03`.
  *(Ngoài ra `validate_labels.py` cũng cảnh báo 10 box lẻ `< 20 px` rải rác ở `img_018`, `img_025`, `img_027`, `img_033`, `img_034`, trong đó `#434` cao `17.4 px` nằm trong mẫu QC `img_034.jpg`.)*
- **Giới hạn mẫu nhỏ:** Với `2` ảnh lỗi trong `7` ảnh mẫu, cận trên khoảng tin cậy 95% (Wilson) của tỉ lệ ảnh lỗi thật lên tới **`64%`**. Mẫu chỉ 7 ảnh quá nhỏ để chứng minh đạt chuẩn `≤ 2%` (ngay cả khi 0/7 ảnh lỗi thì quy tắc 3 vẫn cho thấy lỗi thật có thể tới ~43%).
- **Script kiểm được / không kiểm được:** Các script kiểm tra tốt tính hợp lệ COCO, lớp thuộc taxonomy, box nằm trong kích thước ảnh, box `< 20 px`, tỉ lệ khớp/bắt bẫy so với `traps_key.json` và cận trên Wilson. Tuy nhiên script **không kiểm được** độ ôm sát biên thật của từng box, sai lớp thực tế ngoài bẫy, hay các vật mà **cả máy lẫn người duyệt cùng bỏ sót** (nên mới lọt `missing` trên `img_003`, `img_029`).

**Quyết định: Rework**, ngày `2026-11-18`. **Lý do (theo cây quyết định Hold → Rework → Release):** Không chọn *Hold* vì toàn bộ câu hỏi mở cần thiết (`CAL-01`..`CAL-03`) đã được chốt rõ trong `guideline_v1.1_calibrated.md` và dữ liệu 24 ảnh rõ nét; nhưng bắt buộc phải **Rework** vì (1) tỉ lệ ảnh có lỗi ở mẫu QC là `2/7 = 28.6% > 2%` và (2) tỉ lệ bắt bẫy chỉ đạt `6/12 = 50.0% < 80%` (lọt 3/3 `missing` và 2/3 `loose`), nên lô nhãn hiện tại chưa đủ độ tin cậy để Release.
**Việc cần làm tiếp khi Rework:** (1) Xóa dứt điểm các box thừa `#115` (`img_010.jpg`), `#253` (`img_022.jpg`) và xử lý/lọc bỏ 10 box `< 20 px` (`img_018`, `img_025`, `img_027`, `img_033`, `img_034`); (2) Duyệt lại toàn bộ 24 ảnh trên CVAT theo đúng 3 quy tắc chống thiên kiến ở Mục 4 (ẩn nhãn 15 giây đầu, giữ tốc độ ≥ 2.5 phút/ảnh, rà sát biên 4 cạnh bao gồm mái che xích lô/tuk-tuk); (3) Xuất `qc_fixed.json`, chạy lại `build_release.py` và lấy mẫu QC vòng 2 kiểm chứng.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài:** Có sử dụng trợ lý AI lập trình trong môi trường IDE (Antigravity) để: (a) thiết lập môi trường Python 3.11 `.venv` với PyTorch CUDA 12.4 cho RTX 3050 Ti (tránh xung đột Python 3.14 trên CachyOS), (b) sửa lỗi cấu hình CVAT khi Task `#16` tạo ngoài Project #2, (c) chạy tuần tự các script `prelabel.py`, `threshold_sweep.py`, `merge_prelabels.py`, `inject_traps.py`, `sam2_points.py`, viết script hậu kỳ `work/after_cvat_review.py` tự động trích xuất thời gian thực từ CVAT ClickHouse, và hỗ trợ định dạng bảng số liệu vào Markdown/CSV từ các quyết định và nhận xét của cá nhân tôi. **Cách kiểm tra:** Tôi trực tiếp gán tay tập gold, trực tiếp duyệt 24 ảnh batch trên CVAT, mở mắt đối chiếu ảnh `viz/`, 3 ảnh mask trong `submission/masks/` và 7 ảnh mẫu trong `work/qc/review/*.jpg` để kiểm chứng toàn bộ số liệu trước khi chốt.
- **Gold có bị nhiễm không:** Không. Toàn bộ 4 ảnh `gold_manual.json` (35 đối tượng) được tôi gán tay hoàn toàn độc lập trên CVAT Job `#15` trước khi chạy bất kỳ mô hình AI pre-label nào.
- **Máy / GPU / CVAT dùng; vướng mắc cài đặt:** Máy chạy **CachyOS Linux x86_64**, GPU **NVIDIA GeForce RTX 3050 Ti Laptop GPU (4 GB VRAM, CUDA 12.4)**, **CVAT Docker v2.82.0** tại `http://localhost:8080`. Vướng mắc gặp phải: (1) Python hệ thống mặc định là `3.14.10` chưa có wheel đầy đủ và các hướng dẫn dùng `python` trực tiếp nếu quên kích hoạt `.venv` trong Fish shell sẽ báo `No module named 'numpy'` — giải quyết bằng `.venv` Python 3.11.17 và cài thêm `openai-clip`; (2) Tạo Task `#16` ở ngoài Project gây lỗi `Label 'motorbike' is not registered` — đã gắn lại vào Project `AI-Assisted-Labeling` (ID 2).
- **Dùng cache `data/cache/` thay vì tự chạy?** Không; toàn bộ model YOLO26m, YOLOE-26s và SAM 2.1 Base đều chạy inference trực tiếp trên GPU RTX 3050 Ti của máy.
- **Lỗi / bug script đã gặp:** Khi chạy `review_stats.py` trên `img_010.jpg`, người duyệt đã vẽ bù lại box xích lô `#114` nhưng thiếu phần mái che phía trên khiến `IoU = 0.472 < 0.50` với box gốc nên bị tính là lọt bẫy `missing` — đây là do người vẽ thiếu phần mái che chứ không phải bug script; giữ nguyên số đo trung thực của script và giải trình rõ ở mục 3.
- **Một điều bạn làm khác lần sau:** Không chủ quan lướt nhanh ở bước review (`1.0 phút/ảnh`); bắt buộc ẩn toàn bộ nhãn máy quét kỹ từng góc phần tư ảnh trong 15 giây đầu và kiểm tra đủ biên mái che / kích thước `< 20 px` ngay trong CVAT trước khi export.
