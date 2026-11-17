# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603039 - Đặng Anh Tuyền

> Chỉ ghi phần thay đổi so với v1.0. Mỗi ca có ảnh ví dụ và khớp một dòng trong `qc/calibration_decisions.csv`.
> Câu hỏi mở 1, 2, 5, 6, 7 của v1.0 **chưa chốt**: trong 28 ảnh đã gán/duyệt không gặp ca nào đủ rõ để làm căn cứ (không thấy xe đạp điện; không có ảnh đêm; mask kính xe chưa làm). Để nguyên cho v1.2.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 3 | Hàng nhô ra: box xe có gồm hàng không? | Hàng nằm **trong thùng / giỏ gắn cố định** thuộc box xe. Mẹt, rổ **đặt tạm hoặc treo**, nhô ra ngoài khung xe thì **không** tính vào box xe (box xe dừng ở khung / bánh). Mask áp dụng cùng quy tắc. | img_003 (thùng ba gác chở rau: tính), img_031 (mẹt cam nhô trái: không tính) | CAL-03 |
| 4 | Người ngồi trong / sau vật che: ngưỡng 20% đo thế nào? | Không ước % bằng mắt. Đếm bộ phận: thấy **đầu + ít nhất vai hoặc một chi** → vẽ `person` (box quanh phần nhìn thấy). Chỉ thấy một bàn tay / mảnh áo → không vẽ. | img_006 (bé ngồi bàn giữa, thấy đầu + vai), img_003 (người ngồi sau xe máy, thấy vai + tay + chân) | CAL-04 |

## 2. Ca khó mới (từ lỗi của máy, QC, quy tắc kích thước)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| img_004, img_003, img_014 | Hai box cùng lớp cho cùng một vật | img_004: box YOLOE three_wheeler bị nhân đôi y hệt (cùng conf 0.8149); img_003, img_014: người duyệt vẽ thêm box đè lên box máy đã có | Giữ một box ôm sát hơn, xoá box kia | **4.1.8 (mới):** mỗi vật đúng một box. Hai box cùng lớp IoU > 0.6 coi là trùng. Trước khi export chạy kiểm tra trùng (script), không dựa vào mắt. — CAL-01 |
| img_013, img_002 | Tuk-tuk | YOLO26m: `car` 0.41, box chỉ phủ 2/3 phía trước (img_013); không có box (img_002). YOLOE-26s: `three_wheeler` 0.93 (img_002) | `three_wheeler`, box từ mép mui tới cản sau | **Mục 2 / three_wheeler:** lớp này chỉ lấy từ YOLOE; box `car`/`truck` của YOLO26 chồng lên xe ba bánh → xoá. **4.1.1 bổ sung:** box xe ba bánh phải gồm mui và ghế / thùng sau. Lỗi lớp này **không** sửa được bằng ngưỡng. — CAL-02 |
| img_006 | `ignore_region` (#81) bao một người ngồi xe nhìn thấy được | (vẽ tay khi duyệt) | Vẽ `person` | **Mục 2 / ignore_region nhắc lại:** chỉ dùng cho cụm ≥ 5 vật cao < 20 px không tách được; không dùng cho một vật đơn lẻ. — CAL-04 |
| img_036, img_018 | Box cao 11–19.9 px còn sót sau khi duyệt (7 box gold, 14 box batch) | YOLO26 và cả người gán tay đều vẽ | Xoá | **4.1.6 bổ sung:** chiều cao đo trên ảnh gốc 1024 px; kiểm bằng script (`h < 20`) trước khi export. — CAL-05 |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt câu hỏi mở 3 (hàng nhô ra) và 4 (đo "thấy ≥ 20%" bằng bộ phận cơ thể); thêm 4.1.8 (box trùng); three_wheeler chỉ lấy từ YOLOE và box gồm mui + thùng sau; nhắc lại phạm vi `ignore_region`; kiểm 4.1.6 bằng script | CAL-01 … CAL-05 |
