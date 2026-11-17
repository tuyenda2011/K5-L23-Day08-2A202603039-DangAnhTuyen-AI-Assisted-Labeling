# Báo cáo Day 08 — 2A202603039 - Đặng Anh Tuyền

> Mọi con số kèm mẫu số và nguồn. Nguồn số liệu: `submission/qc/*`, `submission/model_log.csv`, `submission/time_log.csv`, và **CVAT 2.82.1 (Docker) của chính máy này**: bảng `LabeledShape` (trạng thái từng box của Job #20/#21/#22) và analytics ClickHouse (`working_time`, `call:function`). Những chỗ số liệu **không đo được** hoặc **không có giá trị** đều ghi rõ.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26m (COCO) | motorbike 0.35; lớp khác 0.3 (`--conf`) | r02/r03: 0.5 làm mất nhiều vật thật (img_007 person 15 → 6, motorbike 11 → 5), 0.25 thêm nhiều box rác trong đám đông. threshold_sweep (r07) trên gold: motorbike 0.35 cho recall 91% / precision 83% (11 vật). person, car không ngưỡng nào ≤ 0.5 đạt recall 90% (tối đa 84%, 78%) nên giữ mặc định 0.3; bicycle, bus, truck < 5 vật gold nên không chọn ngưỡng riêng |
| three_wheeler | YOLOE-26s, prompt `three_wheeler` | 0.15 | r04: tuk-tuk img_002 0.93 trong khi YOLO26 (r02) không có box nào; r05: YOLO26 gọi tuk-tuk img_013 là `car` 0.41. Sweep: recall 100%, precision 78% ở 0.15 (chỉ 7 vật gold) |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên):
  ```
  python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "motorbike=0.35,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split gold --out work/prelabel/gold_eval/pre.coco.json
  python scripts/merge_prelabels.py submission/prelabel/pre_traps.coco.json work/prelabel/gold_eval/pre.coco.json --conf 0 --split "batch,gold" --out work/prelabel/review_with_gold/pre.coco.json
  ```
  Lệnh merge chỉ dùng để chấm máy trên gold. **Pre-label thật của lô batch (Job #21) được sinh trong CVAT** bằng hàm nuclio `day08-yolo26m-gpu` (chạy cả 24 frame) và `day08-yoloe-gpu` (chỉ chạy 7/24 frame: 0, 1, 4, 6, 7, 9, 13), theo log `call:function`. Ngưỡng tuỳ chọn trong CVAT không được ghi lại, nên **không chắc** lô batch đã dùng đúng chuỗi `--thr` ở trên. Đây là điểm lệch so với recipe.
- **Phiên bản SAM:** SAM 2.1 Base (`sam2.1_b.pt`). Lý do: chạy được trên RTX 4050 Laptop (6 GB) cả offline (r05, 2.13 s/ảnh) lẫn hàm nuclio `day08-sam21-base-gpu` trong CVAT; mask bám biên tốt hơn MobileSAM/SAM 1 bản nhỏ; SAM 3 cần xin quyền tải và nặng hơn. Thư mục `models/` chỉ có bản này.
- **Lỗi không sửa được bằng đổi ngưỡng:** tuk-tuk → `car` (img_013, r05) hoặc không có box (img_002, r02); xe bán tải → `truck` 0.73 (img_002, r02; guideline xếp bán tải vào `car`). Hạ hay nâng ngưỡng chỉ thêm/bớt box, không đổi lớp. Cách sửa: lấy `three_wheeler` từ YOLOE (ưu tiên khi merge), xoá box `car`/`truck` chồng lên xe ba bánh (v1.1, CAL-02), sửa lớp bán tải bằng tay.
- **Điểm cao mà sai:** `truck` 0.73 cho xe bán tải (img_002, r02). **Điểm thấp mà sai:** YOLOE `three_wheeler` 0.37 ×2 trên xe bán tải, 0.40 trên xe đẩy (img_002, r04), tức là ở ngưỡng 0.15 các box này sẽ lọt vào. **Điểm thấp mà đúng:** YOLO26 person 0.27–0.36 trong đám đông img_002 (r02) bị cắt nếu dùng 0.5.
- **Giới hạn:** ngưỡng được chọn và chấm trên **cùng 4 ảnh gold** (47 vật), nên precision/recall của máy trên gold là con số lạc quan (in-sample). three_wheeler chỉ có 7 vật, bicycle 1, bus/truck 0: ngưỡng là ước lượng thô.

## 2. AI error profile

**Bảng trong `qc/review_stats.md` không dùng được.** Nó cho accepted 477/477 = 100%, edited/deleted/added = 0. Lý do: file `pre_traps.coco.json` (18:07) được tạo bằng cách chèn bẫy vào **chính bộ nhãn đã duyệt** `batch_reviewed.json` (17:54), nên "pre" và "final" trùng nhau ngoài vùng bẫy. Script còn loại 28 đối tượng quanh bẫy, và đúng là phần còn lại trùng 100%. Số liệu này không đo được gì về máy.

Profile thật lấy từ trạng thái box trong **Job #21** (duyệt pre-label do CVAT sinh), đọc từ DB của CVAT (`source`: `auto` = box máy giữ nguyên, `semi-auto` = box máy đã sửa, `manual` = vẽ tay):

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 440 | ≤ 440/463 = 95.0% | ≥ 463 box máy (440 + 23 + số đã xoá) |
| edited | 23 | ≤ 23/463 = 5.0% | ≥ 463 box máy |
| deleted | **không đo được chính xác** | — | CVAT không lưu nguồn của box đã xoá; log có 72 lần `delete:object` (lẫn cả box vẽ tay và box của lần chạy model lặp lại) |
| added | 58 (42 vật + 16 `ignore_region`) | 58/521 = 11.1% (vật thật: 42/521 = 8.1%) | 521 đối tượng sau duyệt Job #21 |

Vì không biết số deleted, tỉ lệ accepted/edited chỉ là **cận trên**.

- **Lớp tệ nhất: `three_wheeler`.** 22 box cuối gồm 10 giữ nguyên, 6 sửa, 6 vẽ thêm, tức 12/22 phải can thiệp. Nguyên nhân: YOLOE chỉ được chạy trên 7/24 frame, các tuk-tuk/ba gác ở frame khác phải vẽ tay hoặc sửa từ box `car` của YOLO26. Ngoài ra, ở frame 1 (img_004) YOLOE bị gọi **2 lần** (10:21:01 và 10:21:08 UTC), sinh ra 2 box trùng hệt #24/#26 mà không ai xoá (CAL-01). Lớp car: 9 vẽ thêm + 4 sửa trên 58 box cuối. Ảnh vẽ thêm nhiều nhất: img_024 (13), img_018 (9).
- **So với gold (`review_stats.md` mục 2, dùng được):** precision của **máy** 40/68 = **58.8%** (KTC 95%: 47–70%), recall 40/47 = **85.1%** (KTC 95%: 72–93%), mẫu số 47 đối tượng gold trên 4 ảnh, ở ngưỡng đã chọn. Precision thấp: cứ 10 box máy thì ~4 box thừa/sai. Recall 85%: cứ 7 vật thì máy sót ~1. Lạc quan vì chọn ngưỡng trên chính gold.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- **Bắt bẫy: chưa đo.** Job #22 (`batch-review-traps`, nạp `pre_traps_import.coco.json`) vẫn còn nguyên 507/507 box như lúc nạp, `working_time` = 0 phút. Con số 12/12 = 100% trong `review_stats.md` và `qc_report_auto.md` là **hệ quả của cách tạo file**, không phải kết quả duyệt: bẫy được chèn vào bộ đã duyệt, nên `batch_reviewed.json` tự nhiên "không chứa" bẫy nào. **Nguồn bẫy: chèn vào bộ đã review (Job #21 + bước xoá box < 20 px), không phải vào pre-label gốc.** Mình không báo con số này là bắt bẫy.
- Bằng chứng thiên kiến tự động hóa thay thế, lấy từ QC (mục 7) trên chính lô đã duyệt:
  - **Box trùng lọt qua** (img_003 #1/#4, img_014 #131/#136): người duyệt vẽ thêm box đè lên box máy đã có, tức là không kiểm tra ảnh đã có box hay chưa trước khi vẽ. img_004 #24/#26: box máy nhân đôi y hệt, không bị phát hiện.
  - **Bỏ sót đúng chỗ máy cũng sót**: người bị che sau xe máy (img_003), bé ngồi bàn (img_006). Người duyệt chỉ xét các box máy đưa ra, không quét phần còn lại của ảnh. Đây chính là bước "quét ảnh trước khi nhìn nhãn máy" trong checklist đã bị bỏ.
  - **Bỏ sót box quá nhỏ**: 21 box cao < 20 px (7 gold + 14 batch) qua mắt duyệt, phải xoá bằng script (CAL-05).
- **Accepted cao (≤ 95%) đi cùng QC 4/5 ảnh có lỗi** cho thấy accepted cao không có nghĩa là nhãn tốt. Thời gian duyệt chỉ 1.47 phút/ảnh với ~22 vật/ảnh (khoảng 4 giây/vật), quá nhanh để kiểm từng box và quét chỗ trống.

## 4. Máy có đáng dùng không?

- **Thời gian** (`time_log.csv`, đo bằng `working_time` của CVAT, không bấm giờ tay): gán tay gold 29.4 phút / 4 ảnh = **7.35 phút/ảnh** (59 box, ~0.50 phút/box); duyệt có pre-label 35.3 phút / 24 ảnh = **1.47 phút/ảnh** (521 box, ~0.07 phút/box). Nhanh hơn khoảng 5 lần theo ảnh, 7 lần theo box. Thiên lệch: gán gold trước nên đã quen cảnh và quen guideline; ảnh batch đông hơn (21.7 vật/ảnh so với 14.8 ở gold); và tốc độ duyệt nhanh một phần là do **duyệt chưa kỹ** (mục 7: 4/5 ảnh QC còn lỗi). So sánh này thiên vị cho máy.
- **Kết luận: có, nhưng chỉ cho person / motorbike / car**, dựa trên recall 85.1% (40/47) trên gold và chỉ 23 box máy cần sửa trong Job #21. **Không** dùng YOLO26 cho xe ba bánh. YOLOE chỉ dùng được nếu chạy trên **mọi** ảnh, và precision 78% ở 0.15 nghĩa là còn phải xoá box sai (xe bán tải). Phải giữ thời gian duyệt đủ dài: precision máy 58.8% không cho phép "accept hết".
- **Ba quy tắc giảm thiên kiến** (kiểm tra được bằng số hoặc việc làm):
  1. **Đếm trước, nhìn box sau:** với mỗi ảnh, đếm người/xe theo vùng (trái, giữa, phải) **trước khi** bật lớp nhãn máy, ghi số đếm vào note. Kiểm tra: tổng số đếm khớp số box cuối ±1 trên mẫu QC; QC không còn lỗi `bo_sot` kiểu "máy và người cùng sót".
  2. **Chạy script chặn trước khi export:** phát hiện hai box cùng lớp IoU > 0.6 và box cao < 20 px. Kiểm tra: kết quả phải là **0 trùng, 0 box nhỏ** (lần này là 3 cặp trùng và 21 box nhỏ).
  3. **Không nhận lô khi chưa đo bẫy và thời gian:** duyệt Job #22 và chỉ chấp nhận khi bắt ≥ 10/12 bẫy; đồng thời `working_time` ≥ 3 phút cho ảnh > 20 vật (lần này trung bình 1.47 phút/ảnh). Kiểm tra bằng `review_stats.json` và analytics CVAT.

## 5. Mask SAM cho 3 vật (Part 5)

Dùng `scripts/sam2_points.py` với `sam2.1_b.pt` (mặc định). Số click đếm từ chấm vẽ trên ảnh (xanh = dương, đỏ = âm).

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| img_013 (tuk-tuk) | SAM 2.1 Base | 3 (2 dương: mui, thân; 1 âm: người lái) | **Gồm phần lớn người lái** dù đã click âm lên người lái; gồm cả khách ngồi sau song cửa (áo hồng) | Click âm chỉ bỏ được vài mảng; mép lốm đốm quanh người lái | `sam_img013_tuktuk.jpg` |
| img_003 (ba gác) | SAM 2.1 Base | 3 (2 dương: mui, thùng; 1 âm: mặt đường) | **Gồm người lái, bé ngồi trước và rau trong thùng** | Hàng trong thùng: đúng theo v1.1 (CAL-03). Người: sai, mỗi người phải là vật riêng (v1.0 mục 2, 4.2.4) | `sam_img003_vehicle.jpg` |
| img_010 (người đạp xích lô) | SAM 2.1 Base | 4 (2 dương: áo, quần; 2 âm: bánh sau, mặt đường) | Không gồm hàng; **lấn sang chắn bùn và bánh sau xích lô** (dải đỏ xuống tới y ≈ 860) dù có click âm trên bánh | Mask người tràn sang xe; mép ống quần lốm đốm | `sam_img010_person.jpg` |

- **Đối chiếu guideline 4.2:** 4.2.4 ("hai vật chạm nhau: mỗi vật một polygon, không chồng") là quy tắc **bị SAM vi phạm nhiều nhất**: cả 3 mask đều dính vật bên cạnh (xe ↔ người). 4.2.3 (bóng đổ không thuộc vật) thì SAM làm đúng: mask ba gác không lấy bóng trên đường. **Thiếu:** guideline chưa nói khi mask xe dính người thì sửa thế nào. Đề xuất: mask người làm trước, rồi trừ khỏi mask xe (vùng chạm thuộc vật phía trước). Chưa đưa vào v1.1 vì chưa thử trên đủ ca.

## 6. Calibration (Part 6)

- Số quyết định: **5** (`calibration_decisions.csv`: CAL-01 … CAL-05; 2 `loi_may`, 1 `cau_hoi_mo`, 1 `can_review`/QC, 1 quy tắc kích thước). Quan trọng nhất: **CAL-01** (box trùng), vì nó lọt qua review ở 3/24 ảnh batch và xuất hiện ở 2/5 ảnh QC.
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: 4.1.8 box trùng (IoU > 0.6 cùng lớp); three_wheeler chỉ lấy từ YOLOE và box gồm mui + thùng sau; chốt câu hỏi mở 3 (hàng trong thùng cố định thì tính, mẹt/rổ treo nhô ra thì không) và câu 4 (đo "≥ 20%" bằng bộ phận cơ thể: đầu + vai/chi); `ignore_region` chỉ cho cụm ≥ 5 vật nhỏ; kiểm 4.1.6 bằng script. Câu hỏi mở 1, 2, 5, 6, 7 chưa chốt vì không gặp ca trong dữ liệu.
- **Nguyên nhân gốc:**
  - Do máy: CAL-02 (COCO không có xe ba bánh) và một nửa CAL-01 (YOLOE gọi hai lần).
  - Do **người duyệt sai**, guideline không mơ hồ: phần còn lại của CAL-01 (vẽ đè box có sẵn), CAL-04 (bỏ sót người bị che, dùng `ignore_region` cho một người), CAL-05 (box < 20 px: quy tắc rõ nhưng không kiểm).
  - Do **guideline mơ hồ** thật: CAL-03 (câu hỏi mở 3) và một phần CAL-04 (ước "20%" bằng mắt không tái lập được).
- Lưu ý: nhãn đã nộp **chưa được sửa theo v1.1** (vẫn còn 3 cặp box trùng và các vật bỏ sót ở mục 7). Đây là việc của vòng Rework.

## 7. QC và quyết định (Part 6)

- **Mẫu:** 5 ảnh (`qc_sample.py --n 5 --seed 42`): 0 ảnh `can_review` (không box nào được tick `can_review`, gold lẫn batch) + 5 ảnh ngẫu nhiên: img_003, img_006, img_014, img_031, img_035; 101 đối tượng. Người kiểm: mình (**hạn chế**: người kiểm cũng là người đã duyệt lô).
- Lỗi theo **đối tượng**: **5/101 = 5.0%** · theo **ảnh**: **4/5 = 80%** (ngưỡng Release ≤ 2%).
- **Top lỗi** (`qc_review.csv`):
  - **Thừa** (box trùng): 2. Ví dụ img_003 #1/#4 person, IoU 0.92.
  - **Bỏ sót**: 2. Ví dụ img_006, bé ngồi bàn giữa (~x837, y477).
  - **Sai vị trí/kích thước**: 1. img_031 #432, box người dừng ở hông, chân còn thấy sau bánh xe.
  - Không ghi nhận sai lớp, sai thuộc tính, sai mask.
- **Giới hạn mẫu nhỏ:** 4/5 ảnh lỗi cho cận trên 95% (Wilson) là **96%**. Ngược lại, ngay cả khi 0/5 ảnh lỗi, cận trên vẫn là 43%, nên 5 ảnh **không bao giờ chứng minh được** ngưỡng 2%. Ở đây mẫu nhỏ không cứu được: 4/5 là quá xa ngưỡng.
- **Script kiểm được:** box trùng (IoU), box < 20 px, đủ ảnh/đúng lớp, định dạng. **Không kiểm được:** box có ôm sát không (img_031), vật mà máy và người **cùng** sót (img_003, img_006), người duyệt có làm đúng checklist hay không. Dòng "bắt bẫy 12/12" và "accepted 477/477" trong `qc_report_auto.md` lấy từ `review_stats.json` không hợp lệ (mục 2–3), nên **không** dùng để quyết định.

**Quyết định: Rework**, ngày 2026-11-17. Đi theo cây Hold → Rework → Release:
- **Hold?** Không. Câu hỏi mở 1, 2, 5, 6, 7 chưa chốt nhưng không ca nào trong mẫu QC bị chặn vì thiếu quy tắc; ảnh không hỏng.
- **Rework?** Có:
  - ảnh lỗi 4/5 = 80% > 2%;
  - lỗi lặp lại cùng kiểu (thừa do trùng 2, bỏ sót người bị che 2);
  - lô bẫy chưa được duyệt nên độ tin cậy của người duyệt chưa đo được.

**Việc cần làm tiếp:**
1. Duyệt Job #22 trong CVAT theo checklist (đếm trước, nhìn box sau), export thành `labels/batch_reviewed.json`, chạy lại `review_stats.py`.
2. Áp v1.1: xoá 3 cặp trùng (img_003, img_004, img_014), thêm 2 người bị bỏ sót (img_003, img_006), sửa box img_031 #432, thay `ignore_region` #81 ở img_006 bằng `person`.
3. Chạy script chặn trùng / box < 20 px.
4. Chạy `build_release.py` → `qc_sample.py` (mẫu mới) → `qc_report.py`.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài:** dùng Agents để đọc và so sánh JSON, truy vấn CVAT chỉ đọc, phóng to ảnh và tính IoU khi kiểm QC, chạy script thống kê, soạn nháp báo cáo. Mọi số liệu đã đối chiếu với `submission/qc/*` và CVAT.
- **Gold có bị nhiễm không?** Gold được vẽ tay hoàn toàn trong Job #20 (59/59 box `source = manual`, không gọi hàm model nào trên task gold theo log CVAT). Tuy vậy, mình đã chạy pilot YOLO26/YOLOE (r02–r05, lúc 14:20–14:22) **trước** khi gán gold (Job #20 tạo lúc 14:52), nên đã thấy kiểu box của máy trên các ảnh pilot (không phải ảnh gold). Có thể có nhiễm nhẹ về thói quen vẽ.
- **Máy / GPU / CVAT:** Windows 11 Pro, NVIDIA RTX 4050 Laptop (CUDA), conda env `day08`; CVAT 2.82.1 chạy Docker, tự build 3 hàm nuclio GPU (`day08-yolo26m-gpu`, `day08-yoloe-gpu`, `day08-sam21-base-gpu`). Vướng mắc: Python mặc định của Windows là alias Microsoft Store, phải gọi thẳng python của env; console cp1252 lỗi in tiếng Việt, phải đặt `PYTHONIOENCODING=utf-8`.
- **Dùng cache `data/cache/`?** Không: tự chạy mọi model trên GPU (`work/runs.csv`).
- **Lỗi / bug đã gặp:**
  - `prelabel.py` ghi trùng `run_id` r07 cho hai lần chạy khác nhau (YOLO26 và YOLOE).
  - `review_stats.py` không phát hiện được trường hợp `--pre` được tạo từ chính `--final`, nên in 100% accepted mà không cảnh báo.
  - Trong CVAT, gọi lại hàm detector trên cùng frame sẽ nhân đôi box mà không báo.
- **Lần sau làm khác:**
  - Tạo bẫy từ **pre-label gốc** rồi duyệt một lần duy nhất trên job có bẫy, thay vì duyệt Job #21 rồi chèn bẫy vào kết quả đã duyệt.
  - Chạy YOLOE trên **tất cả** ảnh bằng một file pre-label đã merge theo `--thr`, thay vì gọi tay từng frame trong CVAT.
  - Bấm giờ và tick `can_review` ngay khi gặp ca không chắc.
