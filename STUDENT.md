# Thông tin học viên

- Họ và tên: Đặng Anh Tuyền
- MSHV: 2A202603039
- Lớp (L2 / L3 / khác): (chưa điền — học viên tự xác nhận)
- Tài khoản GitHub: tuyenda2011
- Repo bài nộp (URL): https://github.com/tuyenda2011/K5-L23-Day08-2A202603039-DangAnhTuyen-AI-Assisted-Labeling
- Commit chốt bài (hash): commit mới nhất trên nhánh `main` tại thời điểm nộp (xem lịch sử commit)
- Máy: Windows 11 Pro, NVIDIA GeForce RTX 4050 Laptop GPU (CUDA)
- CVAT (Docker): v2.82.1

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 440 / 23 / không đo được (mẫu số: ≥ 463 box máy, từ trạng thái box Job #21 trong CVAT; bảng 477/477 trong `review_stats.md` không hợp lệ vì bẫy chèn vào bộ đã duyệt); added: 58 (mẫu số: 521 đối tượng sau duyệt; 42 vật + 16 ignore_region)
- So với gold: precision / recall của **máy** 40/68 = 58.8% / 40/47 = 85.1% (4 ảnh gold, in-sample)
- Bắt bẫy: chưa đo — Job #22 chưa được duyệt (0 phút); con số 12/12 trong `review_stats.md` là hệ quả cách tạo file, không phải kết quả duyệt
- Thời gian: gán tay 7.35 phút/ảnh (29.4 phút / 4 ảnh); duyệt có pre-label 1.47 phút/ảnh (35.3 phút / 24 ảnh) — đo bằng working_time của CVAT
- Lỗi QC: 80% theo ảnh (4/5), 5.0% theo đối tượng (5/101)
- **Quyết định:** Rework — ảnh lỗi 4/5 > 2%, lỗi lặp (box trùng, bỏ sót người bị che), lô bẫy chưa duyệt
