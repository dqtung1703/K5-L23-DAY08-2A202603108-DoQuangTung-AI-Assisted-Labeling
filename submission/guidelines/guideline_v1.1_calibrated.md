# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603108 - Đỗ Quang Tùng

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của người duyệt. Mỗi ca khó đều dẫn ảnh ví dụ (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 2 | Khối xe máy đỗ dày đặc gần camera: từng xe hay `ignore_region`? | Xe ở gần camera có chiều cao `>= 20 px` và tách được đường biên vẫn vẽ từng xe riêng; chỉ dùng `ignore_region` khi cụm `>= 5` xe có chiều cao `< 20 px` hoặc bị che chồng không tách được cá thể. | `img_018.jpg` | `CAL-04` |
| 3 | Hàng nhô ra `> 20%` (xe đạp / ba gác chở hàng nhô xa): box và mask có gồm hàng không? | **Không** tính hàng hóa rời nhô ra `> ~20%` chiều dài xe vào box hay mask xe (chỉ ôm khung xe và giỏ/thùng gắn cố định). | `img_031.jpg`, `img_003.jpg` | `CAL-02` |
| 5 | Kính chắn gió / cửa sổ xe có thuộc mask xe không? | **Có**, kính chắn gió thuộc mask xe; nhưng khoảng trống nhìn xuyên qua khung xe chiếm `> 5%` diện tích vật phải đục lỗ loại ra khỏi polygon (không tô kín). | `img_013.jpg`, `img_023.jpg` | `CAL-03` |
| 7 | Có đặt ngưỡng `conf` tối thiểu để chấp nhận box máy không cần sửa không? | **Không** đặt ngưỡng `conf` tự động chấp nhận. `conf` chỉ dùng để lọc ưu tiên kiểm tra box thấp điểm trước; mọi box đều phải qua mắt người theo checklist 4 bước. | `img_028.jpg`, `img_029.jpg` | `CAL-04` |

*(Các câu hỏi mở 1, 4, 6 chưa phát sinh đủ ca điển hình trong 24 ảnh batch nên giữ nguyên hướng xử lý tạm `can_review` ở v1.0, sẽ chốt khi mở rộng lô dữ liệu.)*

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_011.jpg`, `img_002.jpg` | Xe ba bánh tuk-tuk và xe bán tải / sạp hàng đẩy tay đứng gần nhau | YOLO26 gọi tuk-tuk là `truck`; YOLOE (`0.15`) gán nhầm đuôi xe bán tải trắng và sạp hàng thành `three_wheeler` | Chỉ gán `three_wheeler` cho phương tiện 3 bánh giao thông (tuk-tuk, xích lô, ba gác); xe 4 bánh là `car`/`truck`, sạp hàng đẩy tay không gán. | Mục 2 (`three_wheeler`, cột "Không gồm") |
| `img_028.jpg`, `img_029.jpg` | Bẫy `extra` và `loose` lọt trong cảnh phố đông xe máy | Máy tạo box ma hoặc box lệch vài pixel trên nền đông | Bắt buộc ẩn nhãn máy và đếm vật theo góc phần tư trước khi duyệt; kiểm tra lại từng box máy có `IoU` sát biên `<= 2 px`. | Quy trình review (bổ sung bước 0 cho ảnh `> 20` vật) |
| `img_013.jpg`, `img_017.jpg` | Mask SAM trên tuk-tuk có người lái và khung hở | `sam2.1_b.pt` dính đầu/tay/dép người lái dù đã có click âm và tô kín lỗ hở xuyên khung xe | Khi dùng SAM cho `three_wheeler`, đặt tối thiểu 2–3 click âm lên người lái và sửa tay polygon nếu lỗ hở `> 5%` diện tích xe. | Mục 4.2.2 & 4.2.4 |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt câu hỏi mở #2, #3, #5, #7; làm rõ ranh giới `three_wheeler` với xe đẩy/sạp hàng và quy tắc click âm/lỗ hở cho mask SAM | Lỗi lặp của YOLO26/YOLOE, bẫy lọt ở `img_028`/`img_029`, lỗi mask SAM ở `img_013`/`img_017` (`CAL-01` đến `CAL-04`) |
