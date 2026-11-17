# Báo cáo Day 08 — 2A202603108 - Đỗ Quang Tùng

> Báo cáo tổng hợp toàn bộ quy trình pre-label, review, calibration và QC cho 36 ảnh phố châu Á. Mọi tỉ lệ đều kèm mẫu số và nguồn file kiểm chứng.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| `person` / `motorbike` | `yolo26-s` | `0.3` (mặc định) | Bắt tốt người và xe máy ở gần/trung bình (`r01`, `r02`); không có ngưỡng `<= 0.5` nào đạt recall `90%` trên gold (tối đa `79%` trên 19 `person`, `71%` trên 14 `motorbike` ở `threshold_sweep.md`) nên giữ mặc định `0.3`. |
| `car` | `yolo26-s` | `0.45` | Trên 8 vật gold đạt recall `100%` (`8/8`) và precision `73%` (`threshold_sweep.md`); giảm box nhiễu ở xa như `car 0.36/0.42` trong `img_016.jpg` (`r01` vs `r02`). |
| `bicycle` / `bus` / `truck` | `yolo26-s` | `0.3` (mặc định) | Số lượng vật trên 4 ảnh gold quá ít (`< 5` vật: `1` bicycle, `1` bus, `0` truck) nên không chọn ngưỡng riêng theo lớp, giữ mặc định `0.3`. |
| `three_wheeler` | `yoloe-s` (`tuk-tuk`, `cyclo`, `cargo tricycle`) | `0.15` | YOLO26 không có lớp này (`r01`: gọi nhầm xe ba bánh ở `img_011` thành `truck`, bỏ sót tuk-tuk ở `img_002`). YOLOE (`r03`) bắt được trên gold đạt recall `100%` (`7/7`) và precision `78%` tại ngưỡng `0.15` (`threshold_sweep.md`). |

- **Lệnh `merge_prelabels.py` đã dùng:**
  `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "car=0.45,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM đã chọn:** **`sam2.1_b.pt` (SAM 2.1 base)**. Lý do: cân bằng tốt giữa độ sắc nét của biên mask và tốc độ (~5.07s/ảnh trên CPU ở `r04`), nhẹ (~162 MB), không bị giới hạn quyền truy cập gated như SAM 3 trên HuggingFace.
- **Lỗi không sửa được bằng đổi ngưỡng:** Lỗi **mù/nhầm lớp ngữ nghĩa** do danh mục đóng (closed-set) như việc YOLO26 gọi tuk-tuk là `truck` (`img_011.jpg`, conf `0.36`) hoặc bỏ sót hoàn toàn (`img_002.jpg`). Tăng hay giảm ngưỡng đều không làm YOLO26 biết tên `three_wheeler`; phải sửa bằng cách **ghép thêm model open-vocabulary (YOLOE)** và cho `three_wheeler` ưu tiên ghi đè `car`/`truck` khi chồng `IoU >= 0.6` trong `merge_prelabels.py`.
- **Box điểm cao mà sai / điểm thấp mà đúng:** Ở `img_015.jpg` (`r01`), cùng một chiếc xe đạp nhưng YOLO26 tạo 2 box chồng nhau (`bicycle 0.75` và `bicycle 0.38`). Ngược lại, ở `img_011.jpg` (`r01`/`r02`), xe máy chở thùng hàng Shopee Food có `motorbike 0.40` là box đúng nhưng bị mất khi nâng ngưỡng lên `0.5`, và ở `r03` xe ba bánh mép trái `img_011.jpg` chỉ có `conf 0.28` nhưng đúng lớp.
- **Giới hạn chọn ngưỡng:** Ngưỡng được quét và chọn trên chính 4 ảnh gold (`threshold_sweep.md`), sau đó lại chấm hiệu năng máy trên chính 4 ảnh gold đó ở Part 4 (`review_stats.md`). Do chọn và chấm trên cùng dữ liệu (in-sample) với cỡ mẫu nhỏ, precision/recall của máy trên gold mang tính **lạc quan** và ngưỡng cho lớp ít vật chỉ là ước lượng thô.

## 2. AI error profile (từ `submission/qc/review_stats.md`)

*(Đã loại 26 đối tượng quanh 11 vùng bẫy nhân tạo để phản ánh đúng chất lượng thật của máy trên 24 ảnh batch.)*

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| `accepted` (giữ nguyên) | 379 | `379/442 = 85.7%` | 442 box máy |
| `edited` (sửa) | 20 | `20/442 = 4.5%` | 442 box máy |
| `deleted` (xóa) | 42 | `42/442 = 9.5%` | 442 box máy |
| `added` (vẽ thêm) | 45 | `45/444 = 10.1%` | 444 đối tượng sau duyệt |

*(Trong số `deleted + added` có **11 ca là đổi lớp**, nên số vật máy bỏ sót thật ≈ `45 - 11 = 34` vật).*

- **Lớp nào máy tệ nhất?** **`three_wheeler`** tệ nhất: có `24` box bị `deleted`, `6` box `edited`, chỉ `10` box `accepted` (tỉ lệ xóa rất cao so với 21 đối tượng thật của lớp này trong bản release). Nguyên nhân là ngưỡng `0.15` của YOLOE quá thấp nên sinh nhiều box giả/nhầm lên đuôi xe bán tải hoặc sạp hàng (`img_002.jpg`), đồng thời một số ca xe ba bánh vẫn bị nhầm lớp với `truck`/`motorbike`.
- **So với gold (4 ảnh, 50 đối tượng gold):** `TP = 42`, `FP = 14`, `FN = 8` → Precision = **`75.0%`** (KTC 95%: `62–84%`), Recall = **`84.0%`** (KTC 95%: `71–92%`).

## 3. Thiên kiến tự động hóa và ảnh bẫy

- **Bắt bẫy:** **`9/11 = 81.8%`** (đạt ngưỡng `>= 80%`; `inject_traps.py` cài được 11/12 bẫy do thiếu box đủ điều kiện `wrong_class`). Chi tiết: `missing = 3/3`, `wrong_class = 2/2`, `loose = 2/3` (lọt `img_029.jpg`), `extra = 2/3` (lọt `img_028.jpg`).
- **Vì sao lọt:** Cả hai bẫy lọt (`img_028.jpg` và `img_029.jpg`) đều rơi vào nhóm 6 ảnh phố rất đông xe máy (`005, 018, 021, 024, 028, 029`). Khi gặp cảnh quá nhiều box chồng chéo, người duyệt dễ mệt mỏi, giảm chú ý ở bước 2 (kiểm tra box thừa) và bước 4 (kiểm tra độ ôm sát của từng box nhỏ) trong checklist.
- **Accepted cao vs bắt bẫy:** `accepted = 379/442 = 85.7%` đi kèm bắt bẫy `9/11 = 81.8%` và `added = 45/444 = 10.1%` cho thấy pre-label khá hữu ích ở các vật lớn, nhưng **không được phép** lấy `accepted` cao để chủ quan: máy vẫn sót ~34 vật thật (FN) và nếu lơ là ở ảnh đông người sẽ dễ bỏ lọt box thừa (`extra`) hoặc box lệch nhẹ (`loose`).

## 4. Máy có đáng dùng không?

- **Thời gian (từ `submission/time_log.csv`):**
  - Gán tay 4 ảnh gold (`gold_manual`): `25 phút / 4 ảnh` = **`6.25 phút/ảnh`**.
  - Duyệt 24 ảnh batch có pre-label (`batch_review`): `32 phút / 24 ảnh` = **`1.33 phút/ảnh`** (nhanh hơn **~4.7 lần**, giảm **~78.7%** thời gian mỗi ảnh).
  - **Thiên lệch so sánh:** Do đã gán tay 4 ảnh gold trước nên tôi đã quen với quy tắc guideline và bối cảnh phố châu Á khi bước vào duyệt batch; tốc độ `1.33 phút/ảnh` có một phần từ hiệu ứng quen tay chứ không hoàn toàn do AI.
- **Kết luận:** **Đáng dùng**, đặc biệt cho `person`, `motorbike`, `car` (`accepted` cao và tiết kiệm gần 79% thời gian). Riêng lớp `three_wheeler` chỉ nên dùng YOLOE làm gợi ý ban đầu và bắt buộc kiểm tra kỹ để xóa box nhiễu.
- **Ba quy tắc giảm thiên kiến (kiểm tra được):**
  1. Ở **100% ảnh**, bấm ẩn toàn bộ nhãn máy (biểu tượng con mắt trên CVAT) và tự đếm nhẩm số lượng từng lớp trong ~15 giây trước khi hiện nhãn máy (kiểm tra bằng log thao tác/checklist).
  2. Với các ảnh đông `> 20` đối tượng, chia ảnh thành **4 góc phần tư** để quét lần lượt từ trái sang phải và nghỉ bắt buộc **3–5 phút** sau mỗi **10 ảnh** (ghi dòng `break` trong `time_log.csv`).
  3. Lọc danh sách `Objects` theo `conf` tăng dần để ưu tiên rà kỹ các box `< 0.45`, nhưng vẫn kiểm tra `100%` box có `conf >= 0.8` ở bước kiểm tra độ ôm sát (`<= 2 px`).

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_013.jpg` | `sam2.1_b.pt` | 3 (`2+`, `1-`) | Vẫn dính đầu, tay, ống quần, dép người lái dù đã có 1 click âm | Tô kín khoảng trống và phần người ngồi qua khung hở của tuk-tuk | `tuktuk_img013.jpg` |
| `img_017.jpg` | `sam2.1_b.pt` | 4 (`2+`, `2-`) | Không dính 2 người đi bộ phía trước | Dính nhẹ bánh sau tuk-tuk bên trái, sót biển `TAXI` trên nóc, tô kín lỗ hở xuyên khung xe | `tuktuk_mid_img017.jpg` |
| `img_023.jpg` | `sam2.1_b.pt` | 4 (`2+`, `2-`) | Tách tốt khỏi người áo vàng đứng sát và khách ngồi trong | Gộp kính chắn gió vào mask xe; còn lẹm nhẹ vùng tối dưới gầm | `tuktuk_mid_img023.jpg` |

- **Đối chiếu guideline mục 4.2:** Quy tắc 4.2.4 (hai vật chạm nhau thì vùng chạm thuộc vật phía trước) giúp xác định rõ ranh giới khi đặt click âm tách người đi bộ ở `img_017.jpg` và người áo vàng ở `img_023.jpg`. Tuy nhiên, quy tắc 4.2.2 (bỏ lỗ thủng `> 5%` diện tích) khó tự động đạt bằng SAM point prompt vì SAM có xu hướng tô kín bên trong khung mái tuk-tuk; đồng thời cần bổ sung quy tắc rõ về kính chắn gió (đã chốt trong `v1.1`).
- *(Tuỳ chọn Interactor SAM trong CVAT):* Chưa chạy trực tiếp trong CVAT Docker vì bản Docker mặc định chưa cài serverless Nuclio; đã mô phỏng đầy đủ thao tác click dương/âm qua `scripts/sam2_points.py`.

## 6. Calibration (Part 6)

- **Số quyết định:** **`4` quyết định** (`CAL-01` đến `CAL-04` trong `submission/qc/calibration_decisions.csv`). Quan trọng nhất là **`CAL-01`** (làm rõ ranh giới `three_wheeler` với xe bán tải/sạp hàng để giảm 24 box `deleted` của YOLOE) và **`CAL-02`** (chốt quy tắc hàng hóa nhô `> 20%`).
- **Quy tắc mới đã vào `guideline_v1.1_calibrated.md`:** Đã chốt các câu hỏi mở `#2` (tách cụm xe máy gần camera `>= 20 px`), `#3` (không gồm hàng rời nhô `> 20%`), `#5` (kính chắn gió thuộc mask xe, lỗ hở `> 5%` phải bỏ khỏi polygon), `#7` (không đặt `conf` tự động chấp nhận); bổ sung quy tắc loại sạp hàng đẩy tay khỏi `three_wheeler` và quy tắc click âm/lỗ hở cho SAM.
- **Nguyên nhân gốc:** Kết hợp giữa **guideline v1.0 cố ý để mở** (các ca hàng nhô, kính xe, cụm xe máy gần) và **lỗi lặp của máy** (YOLOE nhận nhầm xe bán tải/sạp hàng, YOLO26 không có lớp `three_wheeler`).

## 7. QC và quyết định (Part 6)

- **Mẫu QC:** `5/24` ảnh batch (`0` ca `can_review` + `5` ảnh ngẫu nhiên với `seed = 42`: `img_003.jpg`, `img_006.jpg`, `img_014.jpg`, `img_031.jpg`, `img_035.jpg`), tổng **`91` đối tượng**. Người kiểm: Đỗ Quang Tùng (**hạn chế:** tự kiểm trên dữ liệu chính mình duyệt nên khó hoàn toàn khách quan).
- **Tỉ lệ lỗi (từ `submission/qc/qc_report_auto.md`):**
  - Lỗi theo **đối tượng**: **`3/91 = 3.3%`**
  - Lỗi theo **ảnh**: **`3/5 = 60.0%`** (vượt xa ngưỡng Release `<= 2%`)
- **Top lỗi phát hiện:**
  1. `bo_sot` (`1` lỗi): `img_003.jpg` — bỏ sót xe ba gác chở trái cây rất lớn ở chính giữa ảnh (`three_wheeler`).
  2. `thua` (`1` lỗi): `img_006.jpg` — box `#78` (`car 0.65`) ở cuối ngõ thực tế không có ô tô rõ ràng.
  3. `sai_vi_tri_kich_thuoc` (`1` lỗi): `img_031.jpg` — box `#390` (`person 0.71`) bị cắt cụt ở hông, thiếu phần chân nhìn thấy bên dưới khung xe đạp.
- **Giới hạn mẫu nhỏ:** Với `3/5` ảnh lỗi, cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật lên tới **`88%`**. Ngay cả khi mẫu 5 ảnh đạt `0` lỗi, theo quy tắc 3 thì tỉ lệ lỗi thật vẫn có thể tới ~`60%`; do đó mẫu 5 ảnh không đủ để chứng minh ngưỡng `<= 2%` cho toàn lô.
- **Script kiểm được / không kiểm được:** Script (`validate_labels.py`, `qc_report.py`) kiểm được schema COCO, tên lớp, box nằm trong ảnh và chiều cao `< 20 px` (ví dụ 6 cảnh báo `< 20 px` ở `img_017`, `img_018`, `img_027`), nhưng **không** tự phát hiện được việc bỏ sót cả một xe ba gác lớn (`img_003.jpg`), box thừa (`img_006.jpg`) hay box cắt cụt chân người (`img_031.jpg`) — những lỗi này bắt buộc phải kiểm bằng mắt người đối chiếu ảnh gốc.

**Quyết định: Rework**, ngày 2026-11-17. Lý do (theo cây Hold → Rework → Release): Guideline đã được calibration lên `v1.1` và dữ liệu ảnh hợp lệ (không Hold), nhưng tỉ lệ ảnh lỗi trên mẫu QC là **`3/5 = 60.0% > 2%`** (kèm 6 cảnh báo box `< 20 px` ở `img_017`, `img_018`, `img_027`) nên không đủ điều kiện Release, bắt buộc **Rework**.
**Việc cần làm tiếp:** Quay lại Task `batch-review` trên CVAT để bổ sung box `three_wheeler` ở `img_003.jpg`, xóa box `#78` ở `img_006.jpg`, kéo lại box `#390` ở `img_031.jpg`, xử lý 6 box `< 20 px` sang `ignore_region`, export `submission/labels/qc_fixed.json`, rồi chạy lại `build_release.py` và `qc_sample.py --force`.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài:** Có sử dụng trợ lý AI (Antigravity) để giải thích quy trình/lệnh PowerShell trên Windows, hỗ trợ chạy các script đo/QC, trực quan hóa mask và tổng hợp cấu trúc báo cáo; mọi số liệu trong báo cáo đều được kiểm tra đối chiếu trực tiếp từ output thực tế của `validate_labels.py`, `threshold_sweep.md`, `review_stats.md`, `qc_report_auto.md` và ảnh gốc trong `work/`.
- **Gold có bị nhiễm không:** **Không.** Toàn bộ 4 ảnh gold được gán tay hoàn toàn trên CVAT (`25 phút`) và export `gold_manual.json` trước khi chạy bất kỳ lệnh `prelabel.py` nào.
- **Máy / GPU / CVAT dùng; vướng mắc:** Windows, không có GPU (chạy CPU `torch 2.15.0+cpu`), CVAT Docker `v2.81.0` chạy tại cổng `9080`. Vướng mắc ban đầu là thiếu thư viện `transformers` trong `.venv` ở thư mục ngoài (`C:\Users\admin\day08\.venv`) và không dùng GPU nên chạy `yolo26s` + `sam2.1_b.pt` mất ~`5.07s/ảnh`, đã khắc phục bằng cách cài bổ sung `requirements.txt` và copy trọng số vào `models/`.
- **Dùng cache `data/cache/`:** **Không.** Tất cả pre-label đều tự chạy trực tiếp bằng `prelabel.py`.
- **Lỗi / bug script đã gặp:** Không có bug script; `inject_traps.py` cảnh báo chỉ cài được `11/12` bẫy do thiếu 1 box đủ điều kiện nhưng vẫn đạt yêu cầu `>= 10` bẫy của `check_submission.py`.
- **Một điều làm khác lần sau:** Trong quá trình duyệt `batch-review`, sẽ kiểm tra kỹ các xe chở hàng/xe ba gác lớn (`img_003.jpg`) và rà lại toàn bộ cảnh báo `< 20 px` trước khi export để tránh phải Rework.
