# Thông tin học viên

- Họ và tên: Nguyễn Khánh Tùng
- MSHV: 2A202603144
- Lớp (L2 / L3 / khác): L2
- Tài khoản GitHub: github.com/sfpcshch
- Repo bài nộp (URL): https://github.com/sfpcshch/K5-L23-Day08-2A202603144-NguyenKhanhTung-AI-Assisted-Labeling
- Commit chốt bài (hash): 83bd9af
- Máy: Linux (CachyOS x86_64), GPU NVIDIA GeForce RTX 3050 Ti Laptop GPU (CUDA 12.4)
- CVAT (Docker): v2.82.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 466/502 (92.8%) / 3/502 (0.6%) / 33/502 (6.6%) (mẫu số: 502 box máy); added: 9/478 (1.9%) (mẫu số: 478 đối tượng sau duyệt)
- So với gold: precision / recall của **máy** 21/69 = 30.4% (KTC 95%: 21–42%, mẫu số: 69 box dự đoán) / 21/33 = 63.6% (KTC 95%: 47–78%, mẫu số: 33 đối tượng gold)
- Bắt bẫy: 6/12 = 50.0% (KTC 95%: 25–75%)
- Thời gian: gán tay 5.23 phút/ảnh (20.9 phút / 4 ảnh); duyệt có pre-label 1.00 phút/ảnh (24.1 phút / 24 ảnh)
- Lỗi QC: 28.6% theo ảnh (2/7), 1.7% theo đối tượng (2/120)
- **Quyết định:** Rework — lý do một dòng: Lỗi ảnh mẫu QC 2/7 (28.6%) > 2% và bắt bẫy 6/12 (50.0%) < 80%, cần xóa box thừa (#115, #253), xử lý box < 20px và duyệt lại 24 ảnh theo checklist ẩn nhãn trước.
