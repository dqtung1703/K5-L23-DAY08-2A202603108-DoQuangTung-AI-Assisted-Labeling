# Thông tin học viên

- Họ và tên: Đỗ Quang Tùng
- MSHV: 2A202603108
- Lớp (L2 / L3 / khác): L2A
- Tài khoản GitHub: dqtung1703
- Repo bài nộp (URL): https://github.com/dqtung1703/K5-L23-DAY08-2A202603108-DoQuangTung-AI-Assisted-Labeling
- Commit chốt bài (hash): cập nhật ở commit cuối trên nhánh main
- Máy: Windows, không có GPU (CPU)
- CVAT (Docker): v2.81.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 379 / 20 / 42 (mẫu số: 442 box máy); added: 45 (mẫu số: 444 đối tượng)
- So với gold: precision / recall của **máy** 75.0% / 84.0% (mẫu số: 50 đối tượng gold)
- Bắt bẫy: 9/11 = 81.8%
- Thời gian: gán tay 6.25 phút/ảnh (25 phút / 4 ảnh); duyệt có pre-label 1.33 phút/ảnh (32 phút / 24 ảnh)
- Lỗi QC: 60.0% theo ảnh (3/5), 3.3% theo đối tượng (3/91)
- **Quyết định:** Rework — lý do một dòng: Tỉ lệ ảnh lỗi QC 3/5 = 60.0% > ngưỡng 2% (sót xe ba gác ở img_003, thừa box ở img_006, cắt cụt box ở img_031), cần sửa lỗi trước khi release.
