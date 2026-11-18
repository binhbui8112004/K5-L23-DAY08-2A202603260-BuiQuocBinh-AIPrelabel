# Thông tin học viên

- Họ và tên: Bùi Quốc Bình
- MSHV: 2A202603260
- Lớp (L2 / L3 / khác): L23
- Tài khoản GitHub: binhbui8112004
- Repo bài nộp (URL): https://github.com/binhbui8112004/K5-L23-DAY08-2A202603260-BuiQuocBinh-AIPrelabel
- Commit chốt bài (hash): 50339a0
- Máy: Windows 10/11, không có GPU (chạy CPU)
- CVAT (Docker): CVAT Web (https://app.cvat.ai) - thay cho Docker local do loi cai Docker

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: `493/603 = 81.8%` / `34/603 = 5.6%` / `64/603 = 10.6%` (mẫu số: `603` box máy); added: `59/587 = 10.1%` (mẫu số: `587` đối tượng sau duyệt)
- So với gold: precision / recall của **máy**: `46/63 = 73.0%` (mẫu số: `63` box máy trên gold) / `46/53 = 86.8%` (mẫu số: `53` đối tượng gold)
- Bắt bẫy: `11/12 = 91.7%` (mẫu số: `12` bẫy; đạt vượt ngưỡng 80%, chỉ lọt 1 bẫy `loose` trên `img_024.jpg`)
- Thời gian: gán tay `22/4 = 5.50 phút/ảnh` (`22` phút / `4` ảnh gold); duyệt có pre-label `120/24 = 5.00 phút/ảnh` (`120` phút / `24` ảnh batch)
- Lỗi QC: `60.0%` theo ảnh (`3/5` ảnh mẫu có lỗi), `5.2%` theo đối tượng (`6/115` đối tượng trong mẫu)
- **Quyết định:** `Rework` — lý do một dòng: Dù bắt bẫy đạt `11/12 = 91.7% >= 80%`, tỉ lệ ảnh lỗi trong mẫu QC vẫn là `3/5 = 60.0% > 2%` (chủ yếu do lỗi lặp box bộ phận thừa `5/6` lỗi trên `img_003, img_035`).
