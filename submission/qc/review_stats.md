# Số đo review (det)

- Ảnh tính: 24 (batch,gold); box máy: 603; đối tượng sau duyệt: 587; ngưỡng accepted: IoU >= 0.9
- Đã **loại 28 đối tượng nằm quanh các bẫy** khỏi bảng này: lỗi nhân tạo không phản ánh chất lượng máy.

## 1. Bốn trạng thái (AI error profile)
| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted (giữ nguyên) | 493 | 493/603 = 81.8% | 603 box máy |
| edited (sửa) | 34 | 34/603 = 5.6% | 603 box máy |
| deleted (xoá) | 64 | 64/603 = 10.6% | 603 box máy |
| added (vẽ thêm) | 59 | 59/587 = 10.1% | 587 đối tượng sau duyệt |

Trong số deleted + added, **13 trường hợp là đổi lớp** (máy gán sai lớp, box đúng chỗ): đó không phải máy bỏ sót. Số vật máy **sót thật** ≈ added − đổi lớp = 46.

### Theo lớp
| Lớp | accepted | edited | deleted | added |
|---|---|---|---|---|
| person | 252 | 9 | 9 | 28 |
| bicycle | 5 | 1 | 2 | 6 |
| motorbike | 185 | 16 | 24 | 10 |
| car | 35 | 1 | 2 | 6 |
| bus | 2 | 0 | 1 | 2 |
| truck | 1 | 0 | 4 | 3 |
| three_wheeler | 13 | 7 | 22 | 4 |

## 2. So với gold (nhãn gán tay từ đầu)
Ảnh gold: 4; đối tượng gold: 53 (mẫu số của recall)
| | TP | FP | FN | Precision | Recall |
|---|---|---|---|---|---|
| Máy (pre-label) | 46 | 17 | 7 | 73.0% (KTC 95%: 61–82%) | 86.8% (KTC 95%: 75–93%) |

*Lạc quan: ngưỡng được chọn trên chính các ảnh gold này (in-sample), và chỉ có 4 ảnh. Đọc khoảng tin cậy, đừng đọc con số đơn.*


(Ảnh gold không nằm trong nhãn sau duyệt: không tính residual / new error. Đó là chủ ý: duyệt lại ảnh bạn vừa tự gán tay sẽ đo trí nhớ, không đo thiên kiến.)

## 3. Ảnh bẫy
- Bắt được 11/12 = 91.7% (KTC 95%: 65–99%) (ngưỡng đạt: 80%). Chỉ 12 bẫy nên khoảng tin cậy rất rộng: đọc cùng danh sách bẫy lọt, đừng kết luận chỉ từ %.
  - wrong_class: 3/3
  - missing: 3/3
  - loose: 2/3
  - extra: 3/3
  - lọt: img_024.jpg (loose)

## Gợi ý đọc số
- `deleted` cao → ngưỡng thấp hoặc ánh xạ nhãn sai; `added` cao → máy bỏ sót (xem lớp nào);
- `accepted` thấp (< 50%) → sửa mỗi box lâu hơn vẽ mới: đổi model/ngưỡng hoặc vẽ tay;
- bắt bẫy < 80% → duyệt lại lô, siết checklist (quét ảnh TRƯỚC khi nhìn nhãn máy).
