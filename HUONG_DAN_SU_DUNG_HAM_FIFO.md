# HƯỚNG DẪN SỬ DỤNG HÀM FIFO TRONG EXCELCAPPY
> **Tính giá trị xuất kho và tồn kho theo phương pháp Nhập trước - Xuất trước (FIFO)**  
> Chuẩn mực Kế toán Việt Nam (VAS 02 / Thông tư 200) | Lõi nhị phân Rust Native siêu tốc

---

## 1. TỔNG QUAN & ĐIỂM NỔI BẬT

Hàm `=FIFO()` trong **excelCappy** được xây dựng trên lõi tính toán **Rust Native Core**, giúp giải quyết triệt để các bài toán khó nhất của kế toán kho trong Excel:

- 🚀 **Tốc độ xử lý tức thời:** Tính toán tuyến tính $O(N)$ bằng hàng đợi streaming queue, xử lý mượt mà bảng dữ liệu hàng chục nghìn dòng mà không gây đơ/treo máy.
- ⚡ **Mảng động tràn ô (Dynamic Array Spill 2D):** Viết 1 công thức duy nhất tại ô đầu tiên, kết quả tự động tràn xuống toàn bộ bảng dữ liệu. Không cần copy hay kéo rê công thức từng dòng.
- 🛡️ **Kháng lỗi `#VALUE!` khi tính toán:** Các dòng nhập kho thuần túy tự động trả về số thực `0.0` (thay vì chuỗi rỗng `""`), giúp người dùng cộng, trừ, nhân, chia (`+`, `-`, `*`, `/`) các cột thành tiền thông suốt 100%.
- 📊 **Phân bổ chi tiết lô nhập (Mode 10):** Tự động bóc tách minh bạch từng đồng giá vốn theo từng đợt nhập/số chứng từ gốc, đáp ứng yêu cầu khắt khe của Kế toán trưởng và Kiểm toán độc lập.
- 💡 **Diễn giải gộp theo thang đơn giá:** Diễn giải xuất/tồn tự động gom các lô cùng mức giá mua thành biểu thức ngắn gọn `(SL x Đơn giá)`, sử dụng ký tự `'x'` chuẩn hóa, phối hợp hoàn hảo với hàm `=DIEN_GIAI_KHOI_LUONG()`.
- 🔄 **Xử lý số âm chuẩn VAS 02:**
  - Hàng mua trả lại nhà cung cấp (`SL nhập < 0`): Tự động giảm trừ ngược các lô đã nhập.
  - Hàng bán bị khách trả lại (`SL xuất < 0`): Tự động tái nhập kho theo đúng giá vốn xuất gần nhất.
- ⚠️ **Xử lý linh hoạt khi xuất vượt tồn (âm kho):** Tùy chọn tạm tính theo giá mua gần nhất (`allow_negative = 1`) hoặc chặn báo lỗi `#NUM!` (`allow_negative = 0`).

---

## 2. CÚ PHÁP HÀM `=FIFO()`

```excel
=FIFO(VÙNG_DỮ_LIỆU, [CỘT_MÃ_HÀNG], [CỘT_SL_NHẬP], [CỘT_GIÁ_NHẬP], [CỘT_SL_XUẤT], [CHẾ_ĐỘ], [CỘT_SỐ_LÔ_NGÀY], [XỬ_LÝ_XUẤT_ÂM])
```

### Bảng giải thích chi tiết các tham số:

| STT | Tên tham số | Bắt buộc? | Mặc định | Ý nghĩa & Hướng dẫn thiết lập |
| :---: | :--- | :---: | :---: | :--- |
| **1** | `VÙNG_DỮ_LIỆU` | **Bắt buộc** | — | Vùng bảng dữ liệu nhật ký nhập xuất kho (ví dụ: `$B$8:$R$301`). Khuyến nghị dùng địa chỉ tuyệt đối có dấu `$`. |
| **2** | `CỘT_MÃ_HÀNG` | Tùy chọn | `1` | Thứ tự cột chứa **Mã hàng / Tên hàng** trong vùng dữ liệu (1-based index). |
| **3** | `CỘT_SL_NHẬP` | Tùy chọn | `2` | Thứ tự cột chứa **Số lượng nhập**. |
| **4** | `CỘT_GIÁ_NHẬP`| Tùy chọn | `3` | Thứ tự cột chứa **Đơn giá nhập kho**. |
| **5** | `CỘT_SL_XUẤT` | Tùy chọn | `4` | Thứ tự cột chứa **Số lượng xuất**. |
| **6** | `CHẾ_ĐỘ` | Tùy chọn | `0` | Chế độ kết quả đầu ra (xem bảng chế độ chi tiết bên dưới). |
| **7** | `CỘT_SỐ_LÔ_NGÀY`| Tùy chọn | `0` | Thứ tự cột chứa **Số lô / Ngày chứng từ / Số phiếu** (dùng cho Mode 10). Nếu để `0`, hệ thống tự đánh dấu `Dòng X`. |
| **8** | `XỬ_LÝ_XUẤT_ÂM`| Tùy chọn | `1` | **Chính sách khi xuất kho vượt tồn:**<br>• `1` *(Khuyến nghị)*: Tự động tạm tính theo giá nhập gần nhất, không làm gián đoạn bảng tính.<br>• `0`: Báo lỗi `#NUM!` ngay khi tồn kho bị âm. |

---

## 3. DANH MỤC CÁC CHẾ ĐỘ ĐẦU RA (`CHẾ_ĐỘ`)

| `CHẾ_ĐỘ` | Kiểu Spill | Cột kết quả | Ý nghĩa nghiệp vụ |
| :---: | :---: | :--- | :--- |
| **0** *(Mặc định)* | **$N \times 3$** | **[Đơn giá xuất \| Thành tiền xuất \| Thành tiền tồn]** | **Chế độ đầy đủ nhất:** Tự động điền 3 cột chính của sổ chi tiết kho. Dòng nhập trả về `0.0` để tính toán số học không lỗi. |
| **1** | $N \times 1$ | **[Thành tiền xuất]** | Dùng khi người dùng chỉ cần chèn 1 cột giá trị xuất kho. |
| **2** | $N \times 1$ | **[Diễn giải xuất kho]** | Biểu thức chi tiết các lô bị trừ, ví dụ: `(100 x 50.000) + (20 x 60.000)`. Dòng nhập trả về `""`. |
| **3** | $N \times 1$ | **[Số lượng tồn cuối]** | Lượng hàng còn tồn tại thời điểm sau mỗi dòng phát sinh. |
| **4** | $N \times 1$ | **[Đơn giá bình quân tồn]** | Đơn giá bình quân của số hàng còn tồn kho tại dòng đó. |
| **5** | $N \times 1$ | **[Thành tiền tồn cuối]** | Giá trị tồn kho còn lại sau mỗi dòng phát sinh. |
| **6** | $N \times 1$ | **[Diễn giải tồn kho]** | Biểu thức chi tiết các lô đang tồn theo thang giá, ví dụ: `(28 x 2.585.000) + (8 x 2.531.925)`. |
| **10** | **$M \times 5$** | **[Mã hàng \| Số lô/Ngày \| SL \| Đơn giá \| Thành tiền]** | **Bảng phân bổ chi tiết lô hàng:** Bóc tách toàn bộ các đợt xuất theo từng lô nhập gốc (tương đương hàm `=FIFO_PHAN_BO()`). |

---

## 4. VÍ DỤ ỨNG DỤNG THỰC TẾ

Giả sử bạn có bảng dữ liệu sổ kho nằm ở vùng `$B$11:$O$304` với cấu trúc các cột tính từ cột B (cột 1):
- Cột 1 (Cột B): Tên hàng / Mã hàng
- Cột 2 (Cột C): Ngày hạch toán / Ngày chứng từ
- Cột 8 (Cột I): Số lượng nhập
- Cột 9 (Cột J): Đơn giá nhập
- Cột 11 (Cột L): Số lượng xuất

### Ví dụ 1: Tính sổ chi tiết kho chuẩn mực (Chế độ mặc định Mode 0)
Tại ô đầu tiên của cột Đơn giá xuất (ví dụ ô `M11`), bạn chỉ cần nhập:
```excel
=FIFO($B$11:$O$304, 1, 8, 9, 11)
```
- **Kết quả:** Tự động tràn ra 3 cột:
  - Cột 1 (M): Đơn giá xuất
  - Cột 2 (N): Thành tiền xuất
  - Cột 3 (O): Thành tiền tồn
- **Đặc điểm:** Các dòng chỉ có nghiệp vụ Nhập sẽ hiển thị đơn giá xuất = `0`, thành tiền xuất = `0`, thành tiền tồn = giá trị tồn kho lũy kế. Bạn có thể cộng trừ nhân chia thoải mái mà không bị `#VALUE!`.

---

### Ví dụ 2: Lập bảng phân bổ giá vốn phục vụ kiểm toán (Mode 10 hoặc `=FIFO_PHAN_BO`)
Tại một vùng trống bên cạnh (ví dụ ô `S11`), nhập công thức:
```excel
=FIFO($B$11:$O$304, 1, 8, 9, 11, 10, 2)
```
*Hoặc sử dụng hàm đồng nghĩa:*
```excel
=FIFO_PHAN_BO($B$11:$O$304, 1, 8, 9, 11, 2)
```
*(Trong đó tham số cuối cùng `2` chỉ định lấy cột C - Ngày hạch toán làm thông tin lô nhập).*

- **Kết quả:** Trả về bảng 5 cột độc lập:
  - Cột 1: Tên hàng
  - Cột 2: Số lô / Ngày nhập gốc
  - Cột 3: SL khấu trừ
  - Cột 4: Đơn giá lô nhập gốc
  - Cột 5: Thành tiền giá vốn tương ứng

---

### Ví dụ 3: Lấy diễn giải giá vốn xuất kho (Mode 2)
Tại ô đầu tiên của cột Diễn giải xuất kho (ví dụ `P11`), nhập:
```excel
=FIFO($B$11:$O$304, 1, 8, 9, 11, 2)
```
- **Kết quả:** Hiển thị chuỗi dạng:
  ```text
  (100 x 50.000) + (20 x 60.000)
  ```
- **Tính toán lại ngay bằng hàm:**
  ```excel
  =DIEN_GIAI_KHOI_LUONG(P11)  --> Trả về 6.200.000
  ```

---

### Ví dụ 4: Lấy diễn giải cơ cấu giá tồn kho (Mode 6)
Tại ô đầu tiên của cột Diễn giải tồn kho (ví dụ `Q11`), nhập:
```excel
=FIFO($B$11:$O$304, 1, 8, 9, 11, 6)
```
- **Kết quả:** Các lô cùng mức giá mua sẽ được tự động gộp lại gọn gàng:
  ```text
  (28 x 2.585.000) + (8 x 2.531.925) + (300 x 2.587.500)
  ```

---

## 5. MỘT SỐ LƯU Ý QUAN TRỌNG

1. **Chuẩn hóa vùng dữ liệu:**
   - Hàm tự động xử lý khoảng trắng thừa (Trim), tự động nhận diện chữ hoa/thường (`SP01` khớp với `  sp01  `), tự động bỏ dấu cách đặc biệt Unicode (`\u00A0`).
2. **Tránh lỗi `#SPILL!` trong Excel:**
   - Đảm bảo các ô tính bên dưới và bên phải của ô đặt công thức hoàn toàn trống, không có chữ, số hay ô trộn (Merge Cells) cản trở mảng động tràn xuống.
3. **Thứ tự sắp xếp dòng:**
   - Dữ liệu trong bảng nhập xuất nên được sắp xếp theo đúng trình tự thời gian phát sinh (từ trên xuống dưới) để đảm bảo nguyên tắc Nhập trước - Xuất trước phản ánh đúng thực tế.
