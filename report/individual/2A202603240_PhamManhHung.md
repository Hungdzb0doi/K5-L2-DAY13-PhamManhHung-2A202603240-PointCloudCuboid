# Báo cáo Day 13 — Phạm Mạnh Hùng, 2A202603240

## 1. Kết luận trước
- **Thông tin bài làm:** Học viên Phạm Mạnh Hùng, MSSV: `2A202603240`. Task CVAT cá nhân: `D13-2A202603240`. Bộ 10 frame được giao: `s01f07`, `s01f49`, `s02f29`, `s02f54`, `s03f08`, `s03f40`, `s04f25`, `s04f50`, `s05f28`, `s05f44`.
- **Môi trường thực hành:** Hệ điều hành Windows 11, trình duyệt Microsoft Edge, CVAT Server v2.74.1, UI v2.74.1. MSSV bạn ngồi cạnh đổi kiểm: `2A202603040`.
- **Đánh giá tổng quan:** Toàn bộ 10 frame đã hoàn thành với tổng số **94 cuboid**, vượt mốc khuyến nghị (60–70 hộp) và mốc tối thiểu (53 hộp). Frame có ít cuboid nhất là `s03f40` (5 cuboid) do đoạn đường vắng và mật độ phương tiện thưa; frame nhiều nhất là `s04f25` (17 cuboid) tại khu vực giao lộ đông đúc. Sau khi tinh chỉnh các hộp, phiếu tự kiểm `selfcheck` đạt tuyệt đối **0/94 cuboid có cảnh báo** (0% lỗi cảnh báo). Bản thân tự tin nhất ở việc căn chỉnh góc hướng (`yaw_deg` bám sát trục dọc thân xe và chỉ đúng đầu xe) cùng độ cao đáy hộp chạm sát mặt đường (`z_bottom ≈ -1.7 m`); phần còn thách thức nhất là ước lượng kích thước các xe ở khoảng cách xa (> 45 m) do điểm quét thưa.
- **Cách dùng CVAT:**
  1. *Đổi góc nhìn:* Kéo chuột trái xoay phối cảnh 3D, chuột phải để dời tâm nhìn, con lăn phóng to/thu nhỏ; nháy đúp chuột vào khoảng trống để đưa góc nhìn về mặc định.
  2. *Vẽ và chỉnh cuboid:* Chọn nhãn (Car, Truck, Bus...), chọn chế độ `Shape` (không dùng Track), di chuột đến cụm điểm rồi nháy đúp để đặt; tinh chỉnh tâm, dài, rộng, yaw trên khung `Top`, chỉnh cao độ và đáy chạm đất trên hai khung chiếu `Side` và `Front`.
  3. *Kiểm tra hướng và camera:* Bật `Appearance` → `Cuboid orientation` để kiểm tra vạch đỏ ngắn (`+X`) hướng về đầu xe; chạy lệnh `python scripts/lab.py preview` để đối chiếu cuboid chiếu lên 7 ảnh camera thực tế.

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ. Ảnh: `t1-s01f07-3d.png`, `t1-s01f07-top.png`, `t1-s01f49-3d.png`, `t1-s01f49-top.png`.

| Frame | Phía trước xe ở cạnh nào của ảnh nhìn từ trên xuống; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| s01f07 | Cạnh dưới ảnh nhìn từ trên xuống (-y); +x chỉ sang trái, +y chỉ về phía sau (cạnh trên) | Đường tương đối phẳng; mặt đường thấp hơn LiDAR khoảng -1.75 m | 77.14 m (xe buýt #12) | Car, Bus | Car #1 ở bên trái (x = 3.57 m, y = 0.43 m); các xe ngược chiều ở làn trái (x từ 9.4 m đến 18.3 m) | Dải điểm phản xạ thưa ven đường phía xa (> 55 m) và tán cây |
| s01f49 | Cạnh dưới ảnh nhìn từ trên xuống (-y); +x chỉ sang trái, +y chỉ về phía sau (cạnh trên) | Đường bằng phẳng; mặt đường thấp hơn LiDAR khoảng -1.80 m | 31.72 m (xe con #5) | Car | Car #2 ở làn bên phải (dist_m = 15.84 m, x = 5.88 m) | Cụm điểm phản xạ tán xạ ven vỉa hè bên phải không rõ hình dạng |

- **3 điều chỉ camera cho biết:**
  1. Màu sắc cụ thể của phương tiện (xe màu đen, trắng, đỏ) và chi tiết thẩm mỹ ngoại thất.
  2. Trạng thái đèn tín hiệu giao thông (xanh/đỏ/vàng), biển báo hiệu và vạch kẻ sơn phân làn đường.
  3. Trạng thái đèn phanh hoặc đèn báo rẽ (xi-nhan) của các xe phía trước.
- **3 điều chỉ LiDAR cho biết:**
  1. Khoảng cách hình học 3D chuẩn xác theo đơn vị mét từ cảm biến tới từng điểm bề mặt vật thể.
  2. Kích thước vật lý thực (dài, rộng, cao tính bằng mét) độc lập với luật xa gần của thấu kính camera.
  3. Tầm nhìn toàn cảnh 360° đồng nhất quanh xe, không bị ảnh hưởng bởi bóng râm, chói sáng hoặc sương mù.

Tám frame lướt:

| Frame | Phía trước ở cạnh nào của ảnh nhìn từ trên xuống | Điều lạ (nếu có) |
|---|---|---|
| s02f29 | Cạnh dưới ảnh nhìn từ trên xuống (-y) | Đường thông thoáng, phương tiện phân bố đều ở khoảng cách trung bình (10–30 m) |
| s02f54 | Cạnh dưới ảnh nhìn từ trên xuống (-y) | Có phương tiện di chuyển nhanh vượt lên ở làn bên cạnh |
| s03f08 | Cạnh dưới ảnh nhìn từ trên xuống (-y) | Xuất hiện xe tải (Truck #5) di chuyển phía trước ở cự ly gần 19.39 m |
| s03f40 | Cạnh dưới ảnh nhìn từ trên xuống (-y) | Mật độ giao thông thưa thớt nhất trong 10 frame, nền đường rộng ít xe |
| s04f25 | Cạnh dưới ảnh nhìn từ trên xuống (-y) | Mật độ phương tiện dày đặc (17 cuboid), nhiều xe nối đuôi tạo bóng che |
| s04f50 | Cạnh dưới ảnh nhìn từ trên xuống (-y) | Nhiều xe dừng chờ, cụm điểm phản xạ rõ nét ở cụm đèn hậu |
| s05f28 | Cạnh dưới ảnh nhìn từ trên xuống (-y) | Nhiều vật cản tĩnh ở lề đường bên cạnh dòng xe di chuyển |
| s05f44 | Cạnh dưới ảnh nhìn từ trên xuống (-y) | Cảnh phức tạp có cả Bus và Truck làn đối diện, nhiều xe đỗ sát vỉa hè |

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh: `t2-a-car.png`, `t2-b-bus-truck.png`, `t2-c-ped-cyclist.png`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | s01f07, idx 1 | Car | Đúng (yaw = 269.3°, chỉ về -y) | Đúng (z_bottom = -1.769 m) | Đã kiểm tra đủ 3 khung nhìn và ảnh preview | Đúng (4.55 m × 1.70 m × 1.48 m) |
| (b) xe buýt hoặc xe tải | s03f08, idx 5 | Truck | Đúng (yaw = 270.0°, chỉ về -y) | Đúng (z_bottom = -1.993 m) | Đã kiểm tra đủ 3 khung nhìn và ảnh preview | Đúng (6.21 m × 2.19 m × 2.09 m) |
| (c) người đi bộ hoặc xe hai bánh | s01f07, không có | Pedestrian | - | - | Đã đối chiếu kỹ 7 camera và cụm điểm | Không có người đi bộ rõ nét trong frame |

- **Vòng hiệu chuẩn:** Hiệu chuẩn giúp đồng bộ cách xác định tâm hình học và quy ước vạch đỏ (`+X`) hướng về đầu xe. Cả lớp thống nhất luôn kiểm tra cao độ đáy ở khung Side/Front để tránh cuboid bị chìm hoặc lơ lửng, đồng thời ưu tiên căn sát cụm điểm nhìn thấy được rồi mới suy luận phần bị che khuất.
- **Vật bỏ qua vì không đủ cơ sở:** Trong 10 frame của tôi, khu vực vỉa hè ở frame `s01f49` có một cụm điểm nhỏ thưa (< 8 điểm) ở khoảng cách > 35 m nghi ngờ là người đi bộ hoặc cột biển báo; do không đủ 15–20% thể tích và không thể phân biệt trên ảnh camera nên tôi đã bỏ qua theo quy tắc `LOW_VISIBILITY` / `UNCERTAIN_CLASS` để đảm bảo chất lượng dữ liệu.

## 4. Bốn hiện tượng chất lượng (Bước 4)
Ảnh: `t4-1-bong-che.png`, `t4-2-thua-o-xa.png`, `t4-3-phan-xa-thap.png`, `t4-4-diem-la.png`.

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che | s04f25 / t4-1-bong-che.png / task D13-2A202603240 | Xe đi trước chắn đường truyền tia LiDAR chiếu tới xe đi sau | Cụm điểm bị khuyết phần đầu xe, chỉ nhận được điểm phản xạ từ đuôi và một cạnh bên | Bật thuộc tính `occluded = 1`, dựng cuboid bao trọn cả phần thân suy luận được theo kích thước chuẩn |
| 2. Thưa ở xa | s01f07 / t4-2-thua-o-xa.png / task D13-2A202603240 | Góc mở giữa các tia phát laser tăng dần theo khoảng cách khiến mật độ điểm suy giảm | Vật thể ở cự ly xa (> 60 m như Bus #12) chỉ nhận được vài đường quét ngang rời rạc | Áp dụng quy tắc 15–20% thể tích: nếu nhận diện được hình khối thì vẽ ước lượng theo kích thước chuẩn, nếu quá thưa thì bỏ qua |
| 3. Phản xạ thấp / ít điểm bất thường | s05f28 / t4-3-phan-xa-thap.png / task D13-2A202603240 | Bề mặt kính xe trong suốt hoặc sơn xe màu tối hấp thụ ánh sáng hồng ngoại | Thân xe bị thủng/khuyết một mảng lớn ở vùng kính chắn gió và cửa kính hông | Ghi nhận là giả thuyết, đối chiếu ảnh camera để xác nhận thân xe, giữ nguyên kích thước chuẩn chứ không bóp nhỏ hộp |
| 4. Điểm lạ | s02f54 / t4-4-diem-la.png / task D13-2A202603240 | Tia laser phản xạ nhiều lần (multi-path) qua gương chiếu hậu hoặc mặt đường ướt | Một vài điểm trôi nổi dưới lòng đường (z < -2.5 m) hoặc lơ lửng trên không | Tuyệt đối không gán thành vật thể, không mở rộng đáy cuboid để chứa các điểm nhiễu này |

## 5. Số từ cuboid của tôi (Bước 5)
Số liệu trích xuất từ `submission/D13-2A202603240-bangA.csv` và `submission/D13-2A202603240-tukiem.csv`:

| Cuboid (frame, idx, class) | z | h | z đáy = z − h/2 | dist_m | r = √(x²+y²+z²) | yaw_deg | Nhận xét (lơ lửng / chìm / chạm đường) |
|---|---|---|---|---|---|---|---|
| s01f07, #1, Car | -1.030 m | 1.478 m | -1.769 m | 3.60 m | 3.74 m | 269.3° | Chạm đường phẳng (-1.77 m so với -1.75 m, lệch 0.02 m) |
| s01f49, #1, Car | -1.016 m | 1.529 m | -1.781 m | 6.93 m | 7.00 m | 269.7° | Chạm đường (-1.78 m, độ cao chuẩn) |
| s03f08, #5, Truck | -0.950 m | 2.087 m | -1.993 m | 19.39 m | 19.41 m | 270.0° | Chạm đường (đoạn dốc nhẹ -1.99 m, bám sát mặt đất) |

- **(a) Cuboid gần nhất và xa nhất:**
  - *Gần nhất:* `s01f07 #1 (Car)`: `dist_m = 3.60 m`, khoảng cách 3D `r = √(3.572² + 0.426² + (-1.030)²) = 3.74 m`.
  - *Xa nhất:* `s01f07 #12 (Bus)`: `dist_m = 77.14 m`, khoảng cách 3D `r = √((-3.256)² + (-77.075)² + (-0.453)²) = 77.15 m`.
- **(b) Kích thước so với Bảng 4.3:**
  - *Xe con (`s01f07 #1`):* $l = 4.55\text{ m}$, $w = 1.70\text{ m}$, $h = 1.48\text{ m}$. So sánh chuẩn Car ($l \in [3.8, 5.3]\text{ m}$, $w \in [1.4, 2.3]\text{ m}$, $h \in [1.2, 2.3]\text{ m}$) hoàn toàn nằm trong khoảng chuẩn của dòng sedan phổ thông.
  - *Xe tải (`s03f08 #5`):* $l = 6.21\text{ m}$, $w = 2.19\text{ m}$, $h = 2.09\text{ m}$. So sánh chuẩn Truck ($l \in [4.5, 12.5]\text{ m}$, $w \in [1.9, 2.8]\text{ m}$, $h \in [1.8, 4.2]\text{ m}$) nằm hoàn toàn trong khoảng chuẩn xe tải thùng cỡ trung.
- **(c) Góc yaw của xe cùng chiều và ngược chiều:**
  - Xe cùng chiều phía trước (`s01f07 #1`): `yaw_deg = 269.3°` (rất gần 270°, trục +X hướng thẳng về phía trước theo trục -y).
  - Xe ngược chiều làn đối diện (`s01f07 #7`): `yaw_deg = 92.0°` (rất gần 90°, hướng về phía sau theo trục +y).

Mật độ theo khoảng cách (4 cuboid ở các `dist_m` khác nhau):

| Cuboid (frame, idx, class) | dist_m | Số điểm (`points_in`) |
|---|---|---|
| s01f07, #1, Car | 3.60 m | 3189 |
| s01f07, #2, Car | 19.11 m | 398 |
| s01f07, #6, Car | 29.67 m | 228 |
| s01f07, #3, Car | 48.22 m | 65 |

- **Quy luật mật độ điểm:** Khi khoảng cách tăng lên gấp đôi thì số điểm LiDAR phản xạ từ vật thể giảm đi khoảng 3 đến 4 lần, tuân theo quy luật suy giảm nghịch đảo bình phương khoảng cách ($1/r^2$). Số liệu thực nghiệm trên xe con bám rất sát đường cong lý thuyết trong bài giảng: ở cự ly gần (< 10 m) đạt > 3000 điểm, giảm xuống ~400 điểm ở ~19 m, ~220 điểm ở ~30 m và chỉ còn 65 điểm ở cự ly ~48 m.
- **Tự kiểm và điều chỉnh:**
  - *Các cuboid đã sửa:* Sau lần chạy `sync` đầu tiên, hệ thống ghi nhận 3 cảnh báo kích thước ở `s01f07 #7` (chiều cao), `s01f07 #13` (chiều cao ở xa) và `s05f44 #14` (chiều rộng). Tôi đã mở lại CVAT và tinh chỉnh: hạ chiều cao và nâng đáy xe số 7 chạm mặt đường (height giảm còn 2.27 m, z_bottom = -1.81 m), tăng chiều cao xe số 13 lên 1.20 m, và thu hẹp bề rộng xe số 14 về 2.20 m ôm sát thân xe; đồng thời chỉnh đáy hộp `s03f40 #5` chạm đường. Sau khi Save và chạy lại `sync`, toàn bộ 94/94 cuboid đều đạt chuẩn với 0 cảnh báo.
- **Đổi kiểm chéo:** Đã đổi kiểm chéo số liệu với bạn ngồi cạnh có MSSV: `2A202603040`. Tôi đã đối chiếu trực tiếp trên màn hình của bạn ấy dòng cuboid `s01f00 #1` (Car): hai con số được kiểm tra là khoảng cách `dist_m = 3.34 m` và cao độ đáy `z_bottom = -1.845 m` (tính từ z = -1.045 m, h = 1.600 m: $z_{\text{đáy}} = z - h/2 = -1.045 - 0.800 = -1.845\text{ m}$), kết quả hoàn toàn trùng khớp giữa báo cáo và Bảng A trên máy bạn. Ngược lại, bạn `2A202603040` cũng đã đối chiếu hai giá trị `dist_m = 3.60 m` và `z_bottom = -1.77 m` của cuboid `s01f07 #1` trên máy tôi, các con số đều chuẩn xác.

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
- **(a) Ý nghĩa nhãn 3D đối với mô hình học máy:**
  1. Nhãn 3D cần được đo bằng mét thay vì pixel vì kích thước hình học thực tế của vật thể trong không gian 3 chiều là đại lượng bất biến vật lý, giúp mô hình học được mối quan hệ không gian thực và tính toán cự ly/vận tốc chính xác cho các tác vụ lập quỹ đạo tránh va chạm an toàn của xe tự hành.
  2. Vật thể thưa điểm ở xa bắt buộc phải có quy tắc riêng (quy tắc 15–20% thể tích và đánh dấu LOW_VISIBILITY) nhằm ngăn chặn mô hình học máy bị "ảo giác" (hallucination) hoặc học phải các đặc trưng giả từ nhiễu, từ đó giảm thiểu tối đa tỷ lệ dương tính giả (false positive) trong điều kiện vận hành thực tế.
- **(b) Cách dùng AI:**
  1. *Công cụ:* Trợ lý Antigravity Agent (Gemini).
  2. *Câu hỏi chính:* Hướng dẫn thiết lập môi trường, giải thích hệ trục tọa độ xe VF (x trái, y sau, z lên), và phân tích số liệu tự kiểm tra từ Bảng A.
  3. *Cách dùng kết quả:* Dùng kết quả để hiểu rõ quy chuẩn gán nhãn và đối chiếu số liệu kiểm tra, toàn bộ 94 cuboid đều được tự tay thao tác, căn chỉnh trên giao diện CVAT và chụp ảnh thực tế.
- **(c) Điều còn chưa chắc:**
  - Một số xe chạy ở làn ngược chiều bị che khuất một phần thân bởi dải phân cách cứng giữa đường, việc ước lượng chiều dài phải dựa nhiều vào kích thước trung bình của dòng xe sedan tương đương.
  - Ở khoảng cách trên 50 m, chùm tia laser rất loãng, việc xác định chính xác góc xoay yaw chủ yếu dựa trên giả định hướng di chuyển theo dòng chảy làn đường thay vì hình dạng cụm điểm chi tiết.
