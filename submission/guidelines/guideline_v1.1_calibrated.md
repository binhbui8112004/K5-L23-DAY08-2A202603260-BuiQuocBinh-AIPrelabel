# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603260 - Bùi Quốc Bình

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (hoặc nói rõ câu nào chưa chốt, vì sao)
> và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của bạn. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 2 | Khối xe máy đỗ dày đặc gần camera (không phải ở xa): từng xe hay `ignore_region`? Ngưỡng cụ thể? | Nếu từng xe có chiều cao `>= 20 px` và phần nhìn thấy `>= 20%` diện tích (dù đỗ sát nhau ở tiền cảnh/trung cảnh) thì **bắt buộc gán từng box `motorbike` riêng**. Chỉ dùng `ignore_region` khi cụm `>= 5` xe ở xa có chiều cao `< 20 px` hoặc bị che khuất chồng lấn `> 80%` không thể tách cá thể. | `img_006`, `img_018` | `CAL-02` |
| 3 | Hàng nhô ra: xe đạp / ba gác chở tre, rổ nhô xa: box xe có gồm hàng không? Mask thì sao? | Rổ/thùng gắn cố định vào khung xe hoặc hàng nhô ra `<= 20%` chiều dài xe thì tính vào box/mask xe; phần hàng hóa rời nhô ra `> 20%` chiều dài xe thì **không** kéo box/mask theo phần nhô đó. | `img_003`, `img_031` | `CAL-03` |
| 5 | Kính chắn gió / cửa sổ xe: thuộc mask xe (nhìn xuyên qua thấy nội thất / người) hay không? | **Kính chắn gió thuộc mask của xe**, nhưng **người lái và hành khách ngồi trên xe không thuộc mask xe** — khi dùng SAM phải đặt click âm (`--neg`) lên người lái/hành khách để tách khỏi mask `three_wheeler`. | `img_013`, `img_003` | `CAL-03` |
| 6 | Box máy chồng nhau quá nhiều trong ảnh (đặc biệt trên `three_wheeler` hoặc `person`): gộp hay giữ? | Mỗi thực thể vật lý chỉ giữ **đúng 1 box** ôm toàn bộ phần nhìn thấy hợp lệ; xóa toàn bộ các box con chỉ ôm một bộ phận (như chỉ ôm mái bạt, chỉ ôm thùng dưới của xe ba gác, hoặc chỉ ôm đầu người). | `img_003`, `img_035` | `CAL-01`, `CAL-04` |

*(Các câu hỏi mở 1, 4, 7 của mục 6 v1.0: giữ nguyên quy tắc v1.0 hiện hành — xe hai bánh có bàn đạp tính `bicycle`, có động cơ không bàn đạp tính `motorbike`; giữ ngưỡng che khuất `20%` ở mục 4.1.2 và không dùng `conf` để tự động chấp nhận box mà không nhìn ảnh).*

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_003`, `img_013` | Xe ba gác đẩy hàng có mái bạt (`img_003`) và xe tuk-tuk (`img_013`) | `YOLO26m` gọi tuk-tuk là `car`/`truck`; `YOLOE` phát hiện đúng `three_wheeler` nhưng sinh 3 box chồng nhau trên cùng 1 xe (`#23` cả xe, `#24` thùng xe, `#25` mái bạt trên `img_003`) và bắt nhầm mái hiên ở mép trái/phải (`#6`, `#7`) | Chỉ giữ 1 box `three_wheeler` ôm trọn cả khung xe + mái bạt gắn liền; xóa các box bộ phận trùng lặp và box nhầm ở mép ảnh | Bổ sung mục 2 & 4.1: Không giữ nhiều box `three_wheeler` cho các bộ phận (mái che, thùng) của cùng một xe (`CAL-01`) |
| `img_006`, `img_018` | Hàng xe máy đỗ sát nhau dọc lề đường từ gần đến xa | `YOLO26m` sinh nhiều box chồng lấn; khi duyệt dễ lạm dụng `ignore_region` phủ cả những xe máy to rõ ở gần camera (`#79` trên `img_006`) hoặc để sót box `< 20 px` ở xa (`img_018`) | Trên `img_006`: xóa `ignore_region` ở tiền cảnh, giữ box từng chiếc `motorbike` rõ; trên `img_018`: dùng `ignore_region` bao cụm xe nhỏ `< 20 px` ở xa và xóa các box lẻ `< 20 px` nằm trong cụm | Bổ sung mục 2 & 4.1.6: Phân biệt rõ cụm xe nhỏ ở xa (`< 20 px` → `ignore_region`) và hàng xe đỗ gần (`>= 20 px` → gán từng xe) (`CAL-02`) |
| `img_035`, `img_024`, `img_027` | Các xe máy/người ở trung cảnh có người đứng sát (`img_035`), box lệch (`img_024`), và người nhỏ ở rìa phải (`img_027`) | Bẫy `missing` trên `img_035` (khi vẽ lại box `motorbike` rộng sang người bên cạnh làm `IoU = 0.311 < 0.5`), bẫy `loose` trên `img_024`, bẫy `wrong_class` (`person` → `bicycle`) trên `img_027`, và box `#572` chỉ ôm đầu người đè lên `#578` trên `img_035` | Box `motorbike` chỉ ôm đúng thân xe máy (không gộp người đứng cạnh); rà kỹ các box nhỏ sát mép ảnh để sửa nhãn lớp chứ không xóa nhầm; xóa box con `#572` trùng trong `#578` | Bổ sung mục 4.1.1 & 4.1.4: Ranh giới box xe máy không mở rộng sang người đứng cạnh; cấm giữ box bộ phận nằm lọt trong box toàn thân (`CAL-04`) |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt câu hỏi mở #2, #3, #5, #6; bổ sung quy tắc xóa box bộ phận trùng lặp trên `three_wheeler`/`person`, quy tắc tách mask người lái khỏi xe bằng click âm SAM, và ranh giới dùng `ignore_region` cho hàng xe máy đỗ gần/xa | Khắc phục lỗi lặp của `YOLO26`/`YOLOE` và lỗi phát sinh khi review/QC (`CAL-01`, `CAL-02`, `CAL-03`, `CAL-04`) |
