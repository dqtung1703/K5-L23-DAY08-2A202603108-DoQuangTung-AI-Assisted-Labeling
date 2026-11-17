# Số đo review (det)

- Ảnh tính: 24 (batch,gold); box máy: 442; đối tượng sau duyệt: 444; ngưỡng accepted: IoU >= 0.9
- Đã **loại 26 đối tượng nằm quanh các bẫy** khỏi bảng này: lỗi nhân tạo không phản ánh chất lượng máy.

## 1. Bốn trạng thái (AI error profile)
| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted (giữ nguyên) | 379 | 379/442 = 85.7% | 442 box máy |
| edited (sửa) | 20 | 20/442 = 4.5% | 442 box máy |
| deleted (xoá) | 42 | 42/442 = 9.5% | 442 box máy |
| added (vẽ thêm) | 45 | 45/444 = 10.1% | 444 đối tượng sau duyệt |

Trong số deleted + added, **11 trường hợp là đổi lớp** (máy gán sai lớp, box đúng chỗ): đó không phải máy bỏ sót. Số vật máy **sót thật** ≈ added − đổi lớp = 34.

### Theo lớp
| Lớp | accepted | edited | deleted | added |
|---|---|---|---|---|
| person | 217 | 4 | 6 | 14 |
| bicycle | 3 | 2 | 1 | 4 |
| motorbike | 108 | 8 | 6 | 13 |
| car | 36 | 0 | 3 | 3 |
| bus | 3 | 0 | 0 | 4 |
| truck | 2 | 0 | 2 | 3 |
| three_wheeler | 10 | 6 | 24 | 4 |

## 2. So với gold (nhãn gán tay từ đầu)
Ảnh gold: 4; đối tượng gold: 50 (mẫu số của recall)
| | TP | FP | FN | Precision | Recall |
|---|---|---|---|---|---|
| Máy (pre-label) | 42 | 14 | 8 | 75.0% (KTC 95%: 62–84%) | 84.0% (KTC 95%: 71–92%) |

*Lạc quan: ngưỡng được chọn trên chính các ảnh gold này (in-sample), và chỉ có 4 ảnh. Đọc khoảng tin cậy, đừng đọc con số đơn.*


(Ảnh gold không nằm trong nhãn sau duyệt: không tính residual / new error. Đó là chủ ý: duyệt lại ảnh bạn vừa tự gán tay sẽ đo trí nhớ, không đo thiên kiến.)

## 3. Ảnh bẫy
- Bắt được 9/11 = 81.8% (KTC 95%: 52–95%) (ngưỡng đạt: 80%). Chỉ 11 bẫy nên khoảng tin cậy rất rộng: đọc cùng danh sách bẫy lọt, đừng kết luận chỉ từ %.
  - missing: 3/3
  - loose: 2/3
  - extra: 2/3
  - wrong_class: 2/2
  - lọt: img_029.jpg (loose), img_028.jpg (extra)

## Gợi ý đọc số
- `deleted` cao → ngưỡng thấp hoặc ánh xạ nhãn sai; `added` cao → máy bỏ sót (xem lớp nào);
- `accepted` thấp (< 50%) → sửa mỗi box lâu hơn vẽ mới: đổi model/ngưỡng hoặc vẽ tay;
- bắt bẫy < 80% → duyệt lại lô, siết checklist (quét ảnh TRƯỚC khi nhìn nhãn máy).
