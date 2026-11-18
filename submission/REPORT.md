# Báo cáo Day 08 — 2A202603260 - Bùi Quốc Bình

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| `person`, `bicycle`, `bus`, `truck` | `YOLO26m` (`yolo26m.pt`) | `0.30` (mặc định) | Dẫn `r03`, `r04`, `r08`: `YOLO26m` phát hiện tốt các lớp COCO; trong `threshold_sweep.md`, `person` (tối đa recall 85% trên 20 vật gold) và `bicycle` (tối đa recall 80% trên 5 vật gold) không đạt ngưỡng recall 90% ở `<= 0.5`, còn `bus` (1/53 vật) và `truck` (1/53 vật) có `< 5` vật trong gold nên giữ nguyên ngưỡng mặc định `0.30`. |
| `motorbike` | `YOLO26m` (`yolo26m.pt`) | `0.10` | Dẫn `r03`, `r08` và `submission/qc/threshold_sweep.md`: ở ngưỡng `0.10`, máy đạt recall `92%` (`>= 90%`) và precision `65%` trên `12` vật `motorbike` của tập gold (4 ảnh). |
| `car` | `YOLO26m` (`yolo26m.pt`) | `0.50` | Dẫn `r04`, `r08` và `submission/qc/threshold_sweep.md`: ở ngưỡng `0.50`, máy vẫn giữ được recall `100%` (`7/7` vật gold) với precision `64%`, đồng thời loại bỏ được nhiều box thừa ở xa. |
| `three_wheeler` | `YOLOE-26s-seg` (`yoloe-26s-seg.pt`), prompt: `'tuk-tuk'`, `'cyclo'`, `'cargo tricycle'` | `0.15` | Dẫn `r05`, `r09` và `submission/qc/threshold_sweep.md`: `YOLO26m` không có lớp `three_wheeler` trong 80 lớp COCO; `YOLOE` ở ngưỡng `0.15` đạt recall `100%` (`7/7` vật gold) và precision `78%`. |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên):
  ```powershell
  python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "motorbike=0.1,car=0.5,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json
  ```
- **Phiên bản SAM bạn chọn** (SAM 1 / 2.1 / 3) và lý do: Chọn **SAM 2.1 (`sam2.1_b.pt`)** làm phiên bản chính cho Part 5 (và thử thêm **SAM 1 `mobile_sam.pt`** ở `r07` và `tuktuk_img013_sam1.jpg` để đối chiếu). Lý do: trên CPU, dù `sam2.1_b.pt` chạy chậm hơn `mobile_sam.pt` (`18.54s/ảnh` ở `r06` so với `9.29s/ảnh` ở `r07`), biên mask của `sam2.1_b.pt` liền mạch hơn (`1 polygon`, `317255 px` trên `img_013.jpg`) trong khi `mobile_sam.pt` bị vỡ thành `2 polygon` (`245541 px`) và hụt chi tiết khung xe. Không dùng SAM 3 vì là model gated trên HuggingFace và máy không có GPU rời.
- Lỗi nào **không sửa được bằng đổi ngưỡng** (ví dụ tuk-tuk bị gọi `car`)? Sửa thế nào?
  - Lỗi **thiếu lớp trong từ điển cố định (closed-set taxonomy mismatch)**: `YOLO26m` chỉ biết 80 lớp COCO nên luôn gọi xe tuk-tuk / xích lô / ba gác trên `img_002.jpg`, `img_013.jpg`, `img_032.jpg` là `car` hoặc `truck` dù tăng hay hạ `--conf`.
  - Cách sửa: ghép thêm model open-vocabulary **`YOLOE`** với prompt `'tuk-tuk'`, `'cyclo'`, `'cargo tricycle'` đặt trước `YOLO26m` trong `merge_prelabels.py` (`--iou 0.6`) để box `three_wheeler` của `YOLOE` ghi đè lên box `car`/`truck` trùng vị trí của `YOLO26m`.
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào):
  - **Điểm cao mà sai:** Trên `img_004.jpg` và `img_013.jpg`, `YOLO26m` dự đoán chiếc tuk-tuk là `car` với độ tự tin rất cao (`conf > 0.75–0.85`); hoặc trên `img_003.jpg`, `YOLOE` sinh thêm box `#24 three_wheeler` (`conf = 0.80`) chỉ ôm phần thùng dưới trùng lặp với box `#23` (`conf = 0.90`) ôm cả xe.
  - **Điểm thấp mà đúng:** Trên `img_006.jpg`, chiếc xe máy ở góc trái tiền cảnh `#85 motorbike` chỉ có `conf = 0.21` (nhờ hạ ngưỡng `motorbike=0.1` mới giữ lại được), và trên `img_023.jpg` có chiếc xe ba bánh `three_wheeler` đúng với `conf = 0.18`.
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh) nên số đo máy trên gold là lạc quan:
  - Tập gold chỉ có **4 ảnh (`53` đối tượng hợp lệ)**, trong đó `bus` chỉ có `1/53` vật và `truck` chỉ có `1/53` vật. Việc vừa dùng 4 ảnh gold để chọn ngưỡng (`threshold_sweep.py`) vừa chấm lại precision/recall của máy trên chính 4 ảnh đó (in-sample evaluation) khiến kết quả `Precision = 73.0%`, `Recall = 86.8%` bị **thiên lệch lạc quan (optimistic bias)** và khoảng tin cậy 95% rất rộng.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 493 | `493/603 = 81.8%` | `603 box máy` (sau khi loại 28 đối tượng quanh vùng bẫy) |
| edited | 34 | `34/603 = 5.6%` | `603 box máy` |
| deleted | 64 | `64/603 = 10.6%` | `603 box máy` |
| added | 59 | `59/587 = 10.1%` | `587 đối tượng sau duyệt` |

*(Lưu ý từ `submission/qc/review_stats.md`: Trong số `deleted + added`, có **13 trường hợp là đổi lớp** do máy gán sai lớp nhưng đúng vị trí; do đó số vật máy bỏ sót thật sự là `59 - 13 = 46/587 = 7.8%`).*

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)?
  - Lớp **`three_wheeler`** và **`truck`** là hai lớp máy làm tệ nhất:
    - Với `three_wheeler`: chỉ có `13/42 = 31.0%` box máy được giữ nguyên (`accepted = 13`, `edited = 7`, `deleted = 22`, `added = 4`). Nguyên nhân là ở ngưỡng `0.15`, `YOLOE` thường sinh **nhiều box chồng nhau cho các bộ phận của cùng một chiếc xe ba bánh** (ví dụ trên `img_003.jpg` sinh cả box mái bạt, box thùng xe và box toàn xe) hoặc bắt nhầm mái hiên ở rìa ảnh, buộc người duyệt phải xóa (`deleted = 22`) và chỉnh lại (`edited = 7`).
    - Với `truck`: `accepted = 1/5 = 20.0%`, `deleted = 4/5 = 80.0%`, `added = 3` do `YOLO26m` hay nhầm xe ba gác chở hàng hoặc ô tô tải nhỏ ở xa (`img_003.jpg`, `img_018.jpg`).
- So với gold: precision / recall của **máy** (nguồn `submission/qc/review_stats.md`):
  - **Precision của máy:** `46/63 = 73.0%` (KTC 95%: `61–82%`, với `TP = 46`, `FP = 17`, mẫu số `63` box máy dự đoán trên 4 ảnh gold).
  - **Recall của máy:** `46/53 = 86.8%` (KTC 95%: `75–93%`, với `TP = 46`, `FN = 7`, mẫu số **`53 đối tượng gold`** trên 4 ảnh gold).

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: **`11/12 = 91.7%`** (KTC 95%: `65–99%`, vượt ngưỡng mục tiêu `80% = 10/12`).
  - Phân theo loại bẫy: `extra` bắt được **`3/3`** (`100%`), `wrong_class` bắt được **`3/3`** (`100%`), `missing` bắt được **`3/3`** (`100%`), `loose` bắt được **`2/3`** (`66.7%`).
  - Duy nhất 1 bẫy bị lọt và nguyên nhân cụ thể:
    1. **`img_024.jpg` (`loose`):** Ảnh rất đông đối tượng (`170` box ở ngưỡng thấp); khi dọn các cụm xe máy ở giữa ảnh tôi đã xóa/gộp nhầm box bẫy `motorbike` bị lệch thay vì kéo chỉnh lại cạnh box (bỏ sót bước 4 — kiểm tra và kéo khít biên trước khi xóa).
    2. Ngoài ra, ở vòng rà soát đầu tiên trên CVAT, box `#583 motorbike` vẽ bổ sung trên `img_035.jpg` bị kéo hơi rộng sang người đứng cạnh và box nhỏ `#416` (`6.97 x 21.35 px`) sát rìa phải `img_027.jpg` bị lệch nhẹ khi click chọn; sau khi chuẩn hóa lại đúng ranh giới 2 đối tượng này thì đạt **`11/12 = 91.7%`**.
- Accepted cao (`493/603 = 81.8%`) và bắt bẫy đạt `11/12 = 91.7%` (lọt 1 bẫy `loose` trên ảnh đông `img_024.jpg`) nói gì về cách bạn duyệt?
  - Quy trình ẩn nhãn máy và quét 4 bước đã giúp bắt được `100%` các bẫy `extra` (`3/3`), `missing` (`3/3`) và `wrong_class` (`3/3`). Tuy nhiên trên những ảnh có mật độ phương tiện quá dày đặc như `img_024.jpg`, người duyệt vẫn dễ chịu **thiên kiến tự động hóa (automation bias)** và mệt mỏi thị giác, dẫn tới thao tác xóa nhanh thay vì căn chỉnh lại biên của box bị lỏng (`loose`).

## 4. Máy có đáng dùng không?

- Thời gian (từ `submission/time_log.csv`):
  - **Gán tay từ đầu (`gold_manual`):** `22 phút / 4 ảnh = 5.50 phút/ảnh` (cho `60` đối tượng = `22.0 giây/đối tượng`).
  - **Duyệt có pre-label (`batch_review`):** `120 phút / 24 ảnh = 5.00 phút/ảnh` (cho `604` đối tượng = `11.9 giây/đối tượng`).
  - **Thiên lệch khi so sánh:** 4 ảnh gold được gán tay trước khi duyệt batch nên tôi đã mất thời gian đầu để làm quen công cụ CVAT và đọc `guideline v1.0`; ngược lại, tập batch có nhiều ảnh đông xe máy gấp đôi tập gold (`604/24 = 25.2 đối tượng/ảnh` so với `60/4 = 15.0 đối tượng/ảnh`). Nếu tính theo thời gian trên mỗi đối tượng, pre-label giúp **tiết kiệm `45.9%` thời gian** (`11.9s` so với `22.0s/đối tượng`).
- Kết luận: **Có đáng dùng**, đặc biệt rất hiệu quả cho các lớp `person` (`accepted = 252/270 = 93.3%` box máy), `car` (`accepted = 35/38 = 92.1%`), và `motorbike` (`accepted = 185/225 = 82.2%`). Tuy nhiên với lớp `three_wheeler` (`accepted = 13/42 = 31.0%`), chỉ nên dùng `YOLOE` để gợi ý vị trí rồi người duyệt bắt buộc xóa box bộ phận trùng lặp và chỉnh lại biên.
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. **Ẩn 100% nhãn máy (phím `H` / biểu tượng con mắt trên CVAT) trong 15 giây đầu tiên ở mọi ảnh** và ghi nhẩm số lượng xe ba bánh + người ở tiền cảnh trước khi bật nhãn máy lên (kiểm tra bằng tỉ lệ bắt bẫy `missing >= 90%`).
  2. **Với các ảnh đông `> 30` đối tượng (`img_005, 018, 021, 024, 028, 029`), chia ảnh thành 4 góc phần tư** và bắt buộc rà riêng dải mép trái/phải rộng `100 px` để không bỏ sót hoặc xóa nhầm vật nhỏ sát biên như ca `img_027.jpg` (kiểm tra: `0` lỗi ở vùng biên khi QC).
  3. **Kiểm tra chống trùng lặp bộ phận trên mọi chiếc `three_wheeler` và `person` trước khi bấm `Save`:** mỗi chiếc xe ba bánh chỉ được có đúng `1` box `three_wheeler` (xóa các box chỉ ôm mái bạt hoặc thùng xe có `IoU >= 0.3` hoặc nằm lọt bên trong box chính; kiểm tra tự động bằng script tìm box lồng nhau cùng lớp).

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_013.jpg` (vật 1: xe tuk-tuk chính diện) | `SAM 2.1` (`sam2.1_b.pt`) | `3 click` (`2 pos`: `400,480`, `600,110`; `1 neg`: `720,300`) | Đã tách được người đứng bên phải nhờ click âm `720,300`, nhưng mask (`317255 px = 45.4%` ảnh, `1 polygon`) vẫn **dính một phần người ngồi bên trong khoang lái** nhìn qua khung kính. | Biên ngoài của khung mui và bánh xe rất mượt (`1 polygon`), nhưng vùng hở giữa khung xe có người ngồi bị gộp vào mask xe. | `tuktuk_img013_sam2.jpg` |
| `img_013.jpg` (đối chiếu SAM 1 trên cùng vật 1) | `SAM 1` (`mobile_sam.pt`) | `3 click` (`2 pos`: `400,480`, `600,110`; `1 neg`: `720,300`) | Loại được phần người bên phải tốt hơn nhưng làm **thủng phần thân trên/kính xe**, diện tích giảm còn `245541 px` (`35.2%` ảnh). | Mask bị chia cắt thành **`2 polygon` rời nhau**, biên răng cưa ở vùng chắn bùn và bánh trước. | `tuktuk_img013_sam1.jpg` |
| `img_003.jpg` (vật 2: xe ba gác chở hoa quả có mái bạt) | `SAM 2.1` (`sam2.1_b.pt`) | `4 click` (`2 pos`: `450,480`, `500,160`; `2 neg`: `340,320`, `390,280`) | Nhờ `2 click âm` đặt vào em bé (`340,320`) và người đẩy (`390,280`), mask đã **loại được người ngồi trên xe**, nhưng **vẫn bao gồm toàn bộ khối hoa quả/hàng hóa** trên thùng xe (`114737 px = 16.4%` ảnh, `2 polygon`). | Vì cọc sắt nối mái bạt màuดำ mảnh nên SAM 2.1 tách phần mái bạt phía trên và thùng xe phía dưới thành **`2 polygon` rời nhau**. | `bagac_img003_sam2.jpg` |
| `img_004.jpg` (vật 3: xe tuk-tuk đỗ chéo bị che một phần) | `SAM 2.1` (`sam2.1_b.pt`) | `3 click` (`2 pos`: `650,420`, `620,280`; `1 neg`: `478,350`) | Không dính người đi bộ bên trái nhờ click âm `478,350`; mask (`90862 px = 13.0%` ảnh, `1 polygon`) ôm trọn cả mui bạt, kính chắn gió và thân xe. | Phần bánh sau trong bóng tối hơi lẹm nhẹ vào bóng đổ dưới mặt đường khoảng `3–5 px`. | `tuktuk_img004_sam2.jpg` |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu?
  - **Quy tắc giúp ích:** Mục 4.2.2 (không vẽ lỗ thủng nhỏ `< 5%` diện tích) và mục 4.2.3 (bóng đổ không thuộc vật) giúp đánh giá rõ lỗi lẹm bóng đổ ở `img_004.jpg`.
  - **Quy tắc còn thiếu trong v1.0 (đã bổ sung vào v1.1):** Mục 4.2 của v1.0 chưa quy định rõ khi một chiếc xe ba gác có mái bạt nối bằng thanh sắt mảnh (`img_003.jpg`) khiến SAM sinh ra `2 polygon` rời nhau thì có được giữ multi-polygon hay phải nối qua thanh sắt, và chưa chốt rõ kính chắn gió nhìn xuyên thấy người ngồi trong khoang (`img_013.jpg`) xử lý thế nào.
- *(Tuỳ chọn)* Interactor SAM trong CVAT: Do làm trên CVAT Web (`app.cvat.ai`) bản miễn phí và chạy offline qua `scripts/sam2_points.py`, việc chạy `sam2_points.py` bằng toạ độ cố định cho phép tái lập chính xác `100%` giữa `sam2.1_b.pt` và `mobile_sam.pt` trên cùng toạ độ click, nhưng bất tiện hơn Interactor trực tiếp ở chỗ không vừa nhìn mask vừa click bổ sung theo thời gian thực được.

## 6. Calibration (Part 6)

- Số quyết định: **`4` quyết định** trong `submission/qc/calibration_decisions.csv` (`CAL-01` đến `CAL-04`, gồm đầy đủ các nguồn `loi_may`, `cau_hoi_mo`, `bay_lot`).
  - Quan trọng nhất là **`CAL-01`** (xử lý việc `YOLOE` sinh nhiều box bộ phận trùng lặp trên cùng một chiếc `three_wheeler` ở `img_003.jpg` và `YOLO26m` gán nhầm `car`/`truck` ở `img_013.jpg`) và **`CAL-02`** (chốt ranh giới giữa gán từng xe `motorbike` và dùng `ignore_region` cho hàng xe máy đỗ sát nhau ở tiền cảnh `img_006.jpg` vs hậu cảnh `img_018.jpg`).
- Quy tắc mới đã vào `submission/guidelines/guideline_v1.1_calibrated.md`:
  1. Chốt câu hỏi mở #2 (`CAL-02`): Hàng xe máy đỗ gần camera nếu từng xe cao `>= 20 px` và thấy `>= 20%` diện tích thì bắt buộc gán từng box `motorbike`, cấm dùng `ignore_region` đè lên xe rõ ở tiền cảnh (`img_006.jpg`).
  2. Chốt câu hỏi mở #3 & #5 (`CAL-03`): Hàng hóa rời nhô `> 20%` chiều dài xe không tính vào box/mask xe; kính chắn gió thuộc mask xe nhưng người ngồi trên xe phải dùng click âm (`--neg`) tách khỏi mask xe (`img_003.jpg`, `img_013.jpg`, `img_031.jpg`).
  3. Chốt câu hỏi mở #6 & lỗi máy/bẫy lọt (`CAL-01`, `CAL-04`): Mỗi xe ba bánh hoặc người chỉ giữ `1` box toàn phần duy nhất; xóa toàn bộ các box con chỉ ôm mái bạt, thùng xe (`#24`, `#25` trên `img_003.jpg`) hoặc chỉ ôm đầu người (`#572` trên `img_035.jpg`), và không kéo box `motorbike` lấn sang người đứng cạnh (`img_035.jpg`).
- Nguyên nhân gốc: Kết hợp cả **3 nguyên nhân**:
  1. **Guideline v1.0 để mở** (câu hỏi #2, #3, #5, #6 về hàng xe đỗ gần, hàng nhô ra, kính xe và box chồng nhau).
  2. **Lỗi hệ thống của máy (`YOLOE` / `YOLO26m`)**: `YOLOE` ở ngưỡng thấp `0.15` nhạy với cả từng bộ phận của xe ba gác (mái che, thùng hàng) nên tạo nhiều box trùng lặp.
  3. **Thiên kiến của người duyệt**: sót các box con trùng lặp có điểm tin cậy cao (`#24 conf=0.80` trên `img_003.jpg`) và sót các box `< 20 px` ở `img_018.jpg`, `img_027.jpg`.

## 7. QC và quyết định (Part 6)

- Mẫu: **`5/24` ảnh batch** (`0` ảnh `can_review` + `5` ảnh ngẫu nhiên với `seed = 42`: `img_003.jpg`, `img_006.jpg`, `img_014.jpg`, `img_031.jpg`, `img_035.jpg`), tổng cộng **`115` đối tượng** (nguồn: `submission/qc/qc_sample.csv` và `submission/qc/qc_report_auto.md`). Người kiểm: Bùi Quốc Bình (**hạn chế**: người kiểm đồng thời là người vừa duyệt lô nên dễ lặp lại cùng điểm mù).
- Lỗi theo **đối tượng**: **`6/115 = 5.2%`** (mẫu số: `115` đối tượng trong 5 ảnh mẫu) · Lỗi theo **ảnh**: **`3/5 = 60.0%`** (mẫu số: `5` ảnh trong mẫu; ngưỡng Release yêu cầu `<= 2%`).
- Top lỗi (từ `submission/qc/qc_review.csv` và `submission/qc/qc_report_auto.md`):
  1. **`thua` (`5/6` lỗi = `83.3%` tổng số lỗi):** Ví dụ trên **`img_003.jpg`**, còn sót 2 box con trùng lặp (`#24 three_wheeler` ôm thùng dưới và `#25 three_wheeler` ôm mái bạt đè lên `#23 three_wheeler` ôm cả xe) cùng 2 box bắt nhầm ở sát mép trái/phải (`#6`, `#7`); trên **`img_035.jpg`**, còn sót box `#572 person` chỉ ôm phần đầu nằm lọt trong box toàn thân `#578 person`.
  2. **`sai_vi_tri_kich_thuoc` (`1/6` lỗi = `16.7%` tổng số lỗi):** Ví dụ trên **`img_006.jpg`**, vùng `#79 ignore_region` vẽ quá rộng bao trùm cả hàng xe máy đỗ rất rõ ở tiền cảnh.
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `submission/qc/qc_report_auto.md`):
  - Với cỡ mẫu chỉ **`5` ảnh**, khi phát hiện `3/5` ảnh có lỗi thì **cận trên khoảng tin cậy 95% (Wilson) của tỉ lệ ảnh lỗi thật lên tới `88%`**. Ngay cả khi mẫu 5 ảnh có `0/5` lỗi thì theo quy tắc `3/n`, tỉ lệ ảnh lỗi thật vẫn có thể lên tới `3/5 = 60%`, nên **mẫu nhỏ 5 ảnh hoàn toàn không thể chứng minh được ngưỡng lỗi `<= 2%`** mà chỉ dùng để phát hiện nhanh lỗi hệ thống cần Rework.
- Script kiểm được / không kiểm được:
  - **Kiểm được:** cấu trúc JSON COCO, tên lớp nằm trong taxonomy, số lượng ảnh (`4` gold, `24` batch), cảnh báo box cao `< 20 px` (`validate_labels.py` đã phát hiện `17` box `< 20 px` trên `img_018, img_025, img_027`), thống kê `accepted/edited/deleted/added` và chấm bẫy theo `IoU`.
  - **Không kiểm được:** box có thực sự ôm sát đối tượng hay còn box bộ phận trùng lặp (`#24`, `#25` trên `img_003.jpg` đều hợp lệ về hình học nên `validate_labels.py` báo `OK`, chỉ mắt người khi QC mới thấy là trùng lặp), vật mà cả máy lẫn người cùng bỏ sót ngoài tập gold, hoặc việc vùng `ignore_region` `#79` trên `img_006.jpg` đè lên xe rõ ở tiền cảnh.

**Quyết định: `Rework`**, ngày `2026-11-19`. Lý do (đi theo cây quyết định Hold → Rework → Release):
1. **Không chọn `Hold`** vì dữ liệu 36 ảnh nguyên vẹn và các điểm mơ hồ của guideline v1.0 đã được chốt rõ ràng thành quy tắc trong `guideline_v1.1_calibrated.md`.
2. **Bắt buộc chọn `Rework` (không thể `Release`)** vì dù tỉ lệ bắt bẫy đã đạt **`11/12 = 91.7% >= 80.0%`**, kiểm định mẫu QC cho thấy chất lượng nhãn vẫn chưa đạt ngưỡng phát hành:
   - Tỉ lệ ảnh có lỗi trong mẫu QC là **`3/5 = 60.0% > 2.0%`** (lỗi theo đối tượng là **`6/115 = 5.2%`**).
   - Có **lỗi hệ thống lặp lại**: lỗi box thừa/trùng lặp bộ phận (`thua`) chiếm **`5/6` lỗi**, đồng thời `validate_labels.py` còn cảnh báo `17` box nhỏ `< 20 px` trên `img_018.jpg`, `img_025.jpg`, `img_027.jpg` cần gộp vào `ignore_region`.
- Nếu Rework / Hold, việc cần làm tiếp:
  1. Áp dụng `guideline v1.1` để xóa toàn bộ các box bộ phận trùng lặp trên `three_wheeler` và `person` (đặc biệt trên `img_003.jpg` và `img_035.jpg`).
  2. Thu hẹp vùng `#79 ignore_region` trên `img_006.jpg` về đúng cụm xe mờ ở xa và gán từng chiếc `motorbike` ở tiền cảnh; xóa các box lẻ `< 20 px` trên `img_018.jpg` đã nằm trong `ignore_region`.
  3. Rà lại 3 ảnh bẫy (`img_024.jpg`, `img_027.jpg`, `img_035.jpg`), xuất `submission/labels/qc_fixed.json`, chạy lại `build_release.py` và lấy mẫu QC vòng 2 (`qc_sample.py --n 5 --seed 42 --force`).

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào; "không dùng" nếu không):
  - Có sử dụng trợ lý AI (**Antigravity / Gemini**) để hướng dẫn cài đặt môi trường Python 3.12 (`.venv`), tải trọng số model, giải thích quy trình các lệnh script trong repo và hỗ trợ kiểm tra định dạng bảng biểu báo cáo. Toàn bộ nhãn `gold_manual.json` (4 ảnh) và `batch_reviewed.json` (24 ảnh) được tôi trực tiếp vẽ/duyệt trên giao diện CVAT và kiểm tra lại bằng `validate_labels.py`, `review_stats.py`, cùng ảnh trực quan trong `work/qc/review/`.
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)?
  - **Không.** Tôi đã gán tay hoàn toàn 4 ảnh gold (`img_001.jpg`, `img_012.jpg`, `img_020.jpg`, `img_036.jpg` — tổng cộng `60` đối tượng trong `22` phút) và xuất `gold_manual.json` trước khi chạy bất kỳ model pre-label nào ở Part 2.
- Máy / GPU / CVAT dùng; vướng mắc cài đặt:
  - **Máy:** Windows 10/11, không có GPU rời (chạy hoàn toàn bằng **CPU** với Python 3.12.10 trong `.venv`).
  - **Vướng mắc cài đặt & cách xử lý:**
    1. Máy ban đầu cài Python 3.15 chưa có wheel cho `torch`/`opencv`, đã cài thêm Python 3.12.10 qua `winget` để tạo `.venv`.
    2. Cài đặt Docker Desktop trên Windows gặp lỗi phân quyền dịch vụ ảo hóa nên đã chuyển sang sử dụng **CVAT Web (`https://app.cvat.ai`)** theo luồng Cách A (Offline import/export COCO 1.0).
    3. Lúc mới tạo Task `batch-review` trên CVAT Web do tạo ngoài Project nên gặp lỗi `Label 'motorbike' is not registered for this task`, đã khắc phục bằng cách `Move to project` vào `day08-2A202603260` rồi upload lại `pre_traps_import.coco.json`.
- Dùng cache `data/cache/` thay vì tự chạy?
  - **Không.** Toàn bộ các lượt chạy ở Part 2 (`t_y26`, `t_y26_c05`, `t_yoloe`, `t_sam`, `t_sam1`, `y26_all`, `yoloe_all`) và Part 5 (`sam2_points.py`) đều được chạy trực tiếp bằng CPU trên máy cá nhân (tổng thời gian chạy `y26_all` 36 ảnh là `81.6s` và `yoloe_all` 36 ảnh là `42.8s`, ghi nhận thực tế trong `work/runs.csv`).
- Lỗi / bug script đã gặp (nếu có):
  - Khi tải file export từ CVAT Web về, file tải xuống là gói nén `.zip` chứa `annotations/instances_default.json`; nếu đổi tên trực tiếp file `.zip` thành `.json` thì `validate_labels.py` sẽ báo lỗi `UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe3`. Cần giải nén lấy đúng file `instances_default.json` bên trong.
- Một điều bạn làm khác lần sau:
  - Trước khi kết thúc duyệt mỗi ảnh trên CVAT, tôi sẽ bật chế độ lọc theo từng nhãn (`three_wheeler`, `person`, `motorbike`) để xóa triệt để các box con trùng lặp (như box mái bạt đè lên box xe ba gác trên `img_003.jpg`) và kiểm tra kỹ kích thước box ở rìa ảnh thay vì xóa nhanh.
