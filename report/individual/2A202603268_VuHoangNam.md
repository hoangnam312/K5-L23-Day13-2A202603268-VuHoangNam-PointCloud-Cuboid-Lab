# Báo cáo Day 13 — Vũ Hoàng Nam, 2A202603268

## 1. Kết luận trước

Bài thực hành của tôi sử dụng task `D13-2A202603268` trên CVAT, gồm 10 frame: `s01f13`, `s01f42`, `s02f24`, `s02f42`, `s03f08`, `s03f50`, `s04f18`, `s04f54`, `s05f05` và `s05f31`. Nội dung tập trung vào đọc point cloud, xác định hướng của vật và biểu diễn vật bằng cuboid 3D.

Môi trường thực hiện: Windows, trình duyệt Edge, CVAT 2.84.0.

| Nội dung tổng kết | Kết quả |
|---|---|
| Tổng số cuboid | 72 cuboid |
| Frame ít cuboid nhất | s01f13, s02f42, s03f50, s04f54, s05f05 và s05f31, cùng có 6 cuboid |
| Số cuboid có cảnh báo | 4/72 cuboid, khoảng 5,56% |
| Nội dung nổi bật | 64/72 hộp thuộc class Car; bài tập trung chủ yếu vào xe con |
| Trọng tâm rà soát | Hướng đầu xe, đáy hộp, kích thước và các vật xa; riêng s01f13 cần thêm 2 hộp để đạt yêu cầu 8 hộp |

Số lượng và class theo frame, thống kê từ [bảng A](../../submission/D13-2A202603268-bangA.csv):

| Frame | Số cuboid | Thành phần nhãn |
|---|---:|---|
| s01f13 | 6 | 5 Car, 1 Bus |
| s01f42 | 9 | 9 Car |
| s02f24 | 8 | 7 Car, 1 Pedestrian |
| s02f42 | 6 | 4 Car, 2 Pedestrian |
| s03f08 | 9 | 9 Car |
| s03f50 | 6 | 4 Car, 1 Cyclist, 1 Bus |
| s04f18 | 10 | 9 Car, 1 Cyclist |
| s04f54 | 6 | 6 Car |
| s05f05 | 6 | 5 Car, 1 Truck |
| s05f31 | 6 | 6 Car |
| **Tổng** | **72** | **64 Car, 2 Bus, 1 Truck, 3 Pedestrian, 2 Cyclist** |

Số lượng hộp phản ánh các vật đang được gán nhãn, không phải toàn bộ vật có trong cảnh. Khi lựa chọn vật, ưu tiên vật gần, cụm điểm rõ và nằm trong phạm vi 50 m; những vùng thưa hoặc bị che cần đối chiếu thêm ảnh camera. Bốn cảnh báo lấy từ cột `flags` của [phiếu tự kiểm](../../submission/D13-2A202603268-tukiem.csv); cảnh báo không trực tiếp tính điểm.

Các thao tác chính trong CVAT:

- Kéo chuột trái để xoay khung 3D, chuột phải để di chuyển và con lăn để phóng to hoặc thu nhỏ. Góc nhìn từ trên xuống giúp đọc toàn cảnh quanh xe.
- Vẽ cuboid bằng Shape. Trong Top, chỉnh tâm, chiều dài, chiều rộng và yaw; trong Side và Front, chỉnh độ cao và vị trí đáy. Không xoay hộp trong Side hoặc Front.
- Bật Appearance → Cuboid orientation để hiển thị hướng. Lệnh `preview` tạo ảnh chiếu hộp lên camera, hỗ trợ đối chiếu đầu xe và phạm vi hộp.

## 2. Đọc frame

Hệ tọa độ VF có gốc tại tâm LiDAR: +x sang trái xe, +y về phía sau và +z hướng lên. Phía trước xe là −y. Trên hai ảnh Top hiện có, phía trước nằm bên phải ảnh, +x hướng lên và +y sang trái.

| Frame | Hướng trên ảnh Top | Mặt đường | Khoảng cách tham khảo | Loại vật và phân bố | Vùng cần xem thêm |
|---|---|---|---|---|---|
| s01f13 | Phía trước: phải; +x: lên; +y: trái | Các vệt quét tạo dải tương đối đều trên lòng đường; tài liệu dùng mốc mặt đường khoảng −1,7 m so với LiDAR | Hộp xa nhất trong bảng A: 53,98 m | 5 Car và 1 Bus; xe xuất hiện cả phía trước và phía sau, lệch hai bên trục dọc | Dải điểm ngoài lòng đường và các vùng thưa ở rìa ảnh |
| s01f42 | Phía trước: phải; +x: lên; +y: trái | Mặt đường tạo dải kéo dài theo chiều ngang ảnh; mốc −1,7 m là giá trị tham chiếu | Hộp xa nhất trong bảng A: 50,07 m | 9 Car; nhiều hộp tập trung phía trước, có xe phía sau và ở nhiều vị trí ngang | Các cụm nhỏ ngoài dải xe và vùng phía trước bên phải ảnh |

Các khoảng cách trong bảng là khoảng cách tới tâm cuboid đã gán nhãn, không thay cho phép xác định vật xa nhất nhận ra trên toàn cảnh. Độ dốc và cao độ đường tại từng vị trí cần đọc từ điểm mặt đường.

Ảnh minh họa: `t1-s01f13-3d.png`, `t1-s01f13-top.png`, `t1-s01f42-3d.png` và `t1-s01f42-top.png`. Hai ảnh Top cần thêm chữ “phía trước (−y)”, “+x” và “+y” theo các hướng trên.

Ở góc nghiêng, các cụm điểm thể hiện chiều cao và sự phân bố theo mặt đường. Góc nhìn từ trên xuống thể hiện rõ vị trí các xe quanh LiDAR: s01f13 có hộp xe buýt dài ở phía trước; s01f42 có nhiều hộp xe con nối tiếp và nằm cạnh nhau phía trước. Các vòng quét gần gốc nằm sát nhau hơn các vòng ở xa.

Vai trò của hai nguồn thông tin khi đọc cảnh:

| Nguồn | Thông tin thứ nhất | Thông tin thứ hai | Thông tin thứ ba |
|---|---|---|---|
| Camera — các chi tiết cần đối chiếu trên ảnh gốc | Màu xe và màu bề mặt | Chi tiết đèn, kính và phần đầu/đuôi | Dấu hiệu phân biệt loại xe hoặc tư thế người |
| LiDAR — thông tin hình học thể hiện trong ảnh và tọa độ | Vị trí vật phía trước, phía sau và hai bên | Khoảng cách, kích thước theo mét khi dùng tọa độ | Phân bố điểm, khoảng trống và các vòng quét |

Bảng trên phân biệt vai trò của từng nguồn; bốn ảnh hiện có là ảnh point cloud, không có khung camera để mô tả màu hay chi tiết của từng vật.

Tám frame còn lại được tổng hợp từ bảng A. Với mỗi frame, phía trước luôn là −y trong hệ VF; cạnh tương ứng trên màn hình phụ thuộc góc xoay khi mở ảnh Top.

| Frame | Hướng phía trước | Điểm đáng chú ý từ nhãn hiện có |
|---|---|---|
| s02f24 | −y | 7 xe con và 1 người đi bộ; người đi bộ s02f24/6 cách 8,20 m |
| s02f42 | −y | 4 xe con và 2 người đi bộ; phù hợp đối chiếu hộp người với hộp xe |
| s03f08 | −y | 9 hộp xe con |
| s03f50 | −y | Có cả Car, Cyclist và Bus; Bus s03f50/6 cách 59,79 m, ngoài phạm vi chấm 50 m |
| s04f18 | −y | 10 hộp, nhiều nhất trong bài; gồm 9 Car và 1 Cyclist |
| s04f54 | −y | 6 hộp xe con |
| s05f05 | −y | 5 Car và 1 Truck; Car s05f05/2 cách 3,52 m, gần nhất bảng A |
| s05f31 | −y | 6 hộp xe con; s05f31/6 có cảnh báo chiều cao trong phiếu tự kiểm |

## 3. Ba cuboid có hướng dẫn

Một cuboid gồm bảy tham số: tọa độ tâm x, y, z; chiều dài, rộng, cao; góc yaw. Chiều dài nằm dọc theo hướng yaw. Ba mẫu dưới đây được chọn từ bảng A để đại diện cho xe con, xe buýt và người đi bộ.

| Vật | Frame, idx | Class | Hướng yaw | Đáy và số điểm từ phiếu tự kiểm | Kích thước và đối chiếu bảng 4.3 |
|---|---|---|---|---|---|
| Xe con gần | s01f13, 1 | Car | 269,7°, gần −y | Khoảng hở +0,05 m; 2130 điểm | 4,332 × 1,746 × 1,392 m; dài và rộng trong khoảng sedan/hatchback, cao thấp hơn cận 1,40 m là 0,008 m |
| Xe buýt | s01f13, 3 | Bus | 286,0°, lệch 16,0° so với −y | Khoảng hở +0,02 m; 713 điểm | 12,578 × 2,558 × 3,056 m; so với xe buýt thường, dài vượt cận 12,5 m là 0,078 m |
| Người đi bộ | s02f24, 6 | Pedestrian | 0,0°, hướng +x theo nhãn | Khoảng hở +0,31 m; 224 điểm | 0,678 × 0,524 × 1,564 m; cả ba chiều nằm trong khoảng tham chiếu người lớn |

Mẫu người đi bộ được lấy từ s02f24 vì hai frame đầu hiện chỉ có nhãn Car và Bus; việc dùng frame thay thế cần theo hướng dẫn của coach. Với người đi bộ, yaw cần đối chiếu hướng đi hoặc hướng mặt. Giá trị yaw trong bảng là hướng của hộp đã vẽ.

Các tiêu chí đối chiếu cho cả ba mẫu: Top dùng kiểm tâm, chiều dài, chiều rộng và yaw; Side/Front dùng kiểm chiều cao và đáy; camera dùng phân biệt đầu/đuôi, loại vật và phần bị che. Bảng 4.3 là khoảng tham chiếu, còn hình dạng vật và cụm điểm là căn cứ để chỉnh hộp.

Ảnh minh họa đã lưu: `t2-a-car.png` (s01f13, ID CVAT 5), `t2-b-bus-truck.png` (s01f13, ID CVAT 1), `t2-c-ped-cyclist.png` (s02f24, ID CVAT 22). Cả ba ảnh thể hiện hộp đang chọn, khung 3D, Top/Side/Front, nhãn và kích thước; Cuboid orientation được bật. ID CVAT khác idx trong bảng A.

**Nguyên tắc hiệu chuẩn:** đối chiếu sự khác nhau về tâm, yaw, kích thước và đáy với guideline. Vạch đỏ ngắn của hộp cần chỉ về đầu xe; với người, căn theo hướng đi hoặc hướng mặt. Hướng và đáy được kiểm riêng, tránh chỉ chỉnh hộp theo một góc nhìn. Báo cáo trình bày nguyên tắc này; kết quả trao đổi cụ thể của vòng hiệu chuẩn không được suy ra từ CSV.

**Vật cần cân nhắc bỏ qua:** các cụm rời rạc ở rìa ảnh s01f13 và s01f42 cần đối chiếu camera trước khi xác định loại vật. Nếu chỉ thấy ít điểm, không nhận ra hình dạng hoặc không có cơ sở suy luận phần bị che, áp dụng LOW_VISIBILITY hoặc UNCERTAIN_CLASS theo tình huống; không ép cụm điểm thành một vật. Đây là tiêu chí lựa chọn, không phải danh sách vật đã bỏ qua.

## 4. Bốn hiện tượng chất lượng

Bốn ảnh được chụp trực tiếp trong CVAT local. Bóng che và điểm lạ lấy từ task chung D13-HC485087; hai hiện tượng còn lại lấy từ task cá nhân D13-2A202603268. Nguyên nhân được ghi dưới dạng giả thuyết, tách khỏi biểu hiện nhìn thấy trên point cloud.

| Hiện tượng | Frame / nguồn / ảnh | Biểu hiện và số liệu đối chiếu | Nguyên nhân (giả thuyết) | Cách xử lý theo guideline |
|---|---|---|---|---|
| Bóng che | s02f00, task chung D13-HC485087; t4-1-bong-che.png | Góc nhìn gần từ trên xuống cho thấy vùng ít điểm ngoài phía xe buýt; cam01/cam03 thể hiện xe buýt sát bên trái xe thu dữ liệu | Xe buýt gần chặn tia LiDAR tới vùng phía sau nó | Không coi vùng trống là không có vật; nếu đủ cơ sở suy luận vật bị che thì vẽ phần suy luận được và bật occluded |
| Thưa điểm ở xa | s01f13, Car idx 5, ID CVAT 3, task cá nhân; t4-2-thua-o-xa.png | Xe cách 29,58 m có 139 điểm; ảnh chọn riêng hộp cho thấy các điểm rời và khoảng trống lớn. Cam09 cung cấp bối cảnh phía sau. Car idx 1 ở 5,17 m có 2130 điểm để đối chiếu | Khoảng cách tăng làm các tia quét tách xa nhau trên bề mặt; hướng bề mặt và che khuất cũng ảnh hưởng | Dùng hình dạng còn thấy và camera; áp dụng quy tắc khoảng 15–20% thể tích hoặc có đủ cơ sở suy luận; thiếu cơ sở thì bỏ qua, ghi LOW_VISIBILITY |
| Phản xạ thấp / phân bố điểm không đều | s01f13, Car idx 1, ID CVAT 5, task cá nhân; t4-3-phan-xa-thap.png | Xe gần ở 5,17 m có các dải điểm tập trung ở một phần hộp, trong khi phần còn lại thưa hơn; cam01 và cam06 cho thấy bề mặt xe tối màu và vùng kính. Tổng số điểm trong hộp là 2130, không dùng tổng này để kết luận toàn xe ít điểm | Vùng kính hoặc bề mặt tối có thể phản hồi yếu; góc tới và phần bề mặt quay ra xa LiDAR cũng có thể gây phân bố không đều | Giữ nguyên nhận định nguyên nhân là giả thuyết; đối chiếu hình dạng trên camera và các mặt hộp, không thu hộp chỉ theo vùng có nhiều điểm |
| Điểm lạ | s02f35, task chung D13-HC485087; t4-4-diem-la.png | Ảnh mở rộng khung 3D ở góc gần ngang thể hiện một cụm dạng vệt ngắn tách dưới mặt đường, nằm ở phần dưới bên trái ảnh. Tài liệu gợi ý của lab mô tả cụm khoảng 46 điểm ở z ≈ −13,5 m | Có thể liên quan phản xạ nhiều đường hoặc sai số đo | Kiểm chênh cao so với mặt đường; không gán cụm điểm dưới mặt đường thành một vật |

Nguồn gợi ý vị trí bóng che và điểm lạ: [hướng dẫn hiện tượng](../../docs/PHENOMENA.md). Các số khoảng cách và số điểm của hai xe lấy từ bảng A và phiếu tự kiểm của task cá nhân. Tên ảnh được giữ trong báo cáo; ảnh chỉ nằm trong ZIP nộp LMS.

## 5. Số từ cuboid của tôi

Nguồn số liệu là `submission/D13-2A202603268-bangA.csv` và `submission/D13-2A202603268-tukiem.csv` sau lần đồng bộ cuối. Mỗi cuboid được xác định bằng frame và idx để đối chiếu giữa hai bảng.

Đáy cuboid được tính theo công thức `z_đáy = z − h/2`. Khoảng cách trên mặt phẳng ngang là `dist_m = √(x² + y²)`, còn khoảng cách không gian từ LiDAR tới tâm hộp là `r = √(x² + y² + z²)`. Các đại lượng này có đơn vị mét. Nhận xét về đáy hộp cần dựa vào mặt đường tại vị trí của vật.

| Cuboid (frame, idx, class) | z (m) | h (m) | z đáy (m) | dist_m (m) | r (m) | yaw (°) | Đáy so với mặt đường |
|---|---|---|---|---|---|---|---|
| s01f13, 1, Car | −1,006 | 1,392 | −1,702 | 5,17 | 5,266 | 269,7 | ground_gap_m = +0,05 m |
| s01f13, 2, Car | −0,965 | 1,392 | −1,661 | 18,54 | 18,566 | 269,7 | ground_gap_m = +0,11 m |
| s01f13, 3, Bus | −0,203 | 3,056 | −1,731 | 29,78 | 29,783 | 286,0 | ground_gap_m = +0,02 m |

Ví dụ, Car s01f13/1 có đáy bằng −1,006 − 1,392/2 = −1,702 m. Ba giá trị `ground_gap_m` trong phiếu tự kiểm lần lượt là +0,05 m, +0,11 m và +0,02 m, đều nằm trong ngưỡng 0,3 m dùng để rà soát khoảng hở với mặt đường. Riêng Bus s01f13/3, tính từ các cột đã làm tròn cho kết quả −1,731 m, còn cột `z_bottom` lưu −1,732 m; chênh lệch 0,001 m do làm tròn.

**Khoảng cách gần nhất và xa nhất**

| Trường hợp | Frame, idx, class | dist_m (m) | r (m) |
|---|---|---|---|
| Gần nhất theo dist_m | s05f05, 2, Car | 3,52 | 3,626 |
| Xa nhất theo dist_m | s03f50, 6, Bus | 59,79 | 59,792 |

Bus xa nhất có tâm ngoài phạm vi 50 m được chấm theo đề. Đây vẫn là cuboid xa nhất trong toàn bộ bảng A hiện tại.

**Đối chiếu kích thước với bảng 4.3**

| Vật | Frame, idx | Dài (m) | Rộng (m) | Cao (m) | Chiều ngoài khoảng và giải thích |
|---|---|---|---|---|---|
| Người đi bộ | s02f24, 6 | 0,678 | 0,524 | 1,564 | Cả ba chiều nằm trong khoảng tham chiếu người lớn |
| Xe con | s01f13, 1 | 4,332 | 1,746 | 1,392 | So với nhóm sedan/hatchback, chiều cao thấp hơn cận 1,40 m là 0,008 m; dài và rộng trong khoảng |

Theo [bảng 4.3](../../docs/GUIDELINE.md), người lớn có dài 0,4–0,9 m, rộng 0,5–0,9 m và cao 1,50–1,95 m; sedan/hatchback có dài 3,8–5,0 m, rộng 1,70–1,90 m và cao 1,40–1,60 m. Car s01f13/1 thấp hơn cận tham chiếu 0,008 m, tương đương khoảng 0,57% của 1,40 m. Mức lệch này nhỏ so với biên sai số 5% nêu trong guideline cho xe; tuy nhiên, bảng tham chiếu chỉ hỗ trợ rà soát. Khi chỉnh chiều cao, căn cứ vào đỉnh và đáy của vật trên Side/Front thay vì tăng chiều cao chỉ để khớp khoảng tham chiếu.

**Đối chiếu hướng**

Trong hệ tọa độ VF, xe hướng thẳng cùng chiều xe thu dữ liệu có yaw khoảng 270°, xe hướng thẳng ngược chiều có yaw khoảng 90°.

| Trường hợp | Frame, idx | yaw (°) | Đối chiếu hướng thực tế |
|---|---|---|---|
| Xe có yaw gần hướng cùng chiều, nằm phía trước | s01f13, 1 | 269,7 | Lệch 0,3° so với 270°; tâm y = −4,139 m |
| Xe có yaw gần hướng ngược chiều | s02f24, 3 | 84,8 | Lệch 5,2° so với 90° |

Các giá trị trên mô tả hướng của nhãn đã vẽ; phần đối chiếu mũi tên với đầu xe trên ảnh sẽ bổ sung cùng ảnh minh họa.

**Số điểm theo khoảng cách**

| Cuboid (frame, idx, class) | dist_m (m) | Số điểm trong hộp (points_in) |
|---|---|---|
| s01f13, 1, Car | 5,17 | 2130 |
| s01f13, 2, Car | 18,54 | 424 |
| s01f13, 4, Car | 20,91 | 292 |
| s01f13, 5, Car | 29,58 | 139 |

Trong bốn xe cùng frame này, số điểm trong hộp giảm khi khoảng cách tăng. Từ 5,17 m tới 29,58 m, khoảng cách tăng khoảng 5,72 lần, còn số điểm giảm từ 2130 xuống 139, còn khoảng 6,53%. Đây là số điểm trong hộp, chưa chia cho thể tích. So với các mức trung bình trong slide (1968 điểm ở 0–10 m, 718 ở 10–20 m, 214 ở 20–30 m), các ví dụ có cùng xu hướng giảm; từng vật còn khác nhau về kích thước, hướng nhìn và mức che khuất.

**Nội dung chỉnh sửa cần thực hiện sau khi tự kiểm**

| Frame, idx | Số liệu hiện tại | Hướng chỉnh và tiêu chí đối chiếu |
|---|---|---|
| s02f42, 1 | Car dài 4,738 m, rộng 3,466 m, cao 0,857 m | Kiểm tra hộp có bao sang vật bên cạnh hoặc bị đổi chiều dài/rộng không; căn lại các mặt bên trong Top và kiểm chiều cao trong Side/Front |
| s05f05, 6 | Car cao 0,640 m, đáy −1,023 m; ground_gap_m = +0,69 m | Kiểm tra hộp có chỉ bao phần trên của xe không; căn đáy về mặt đường tại vật và đặt lại chiều cao theo điểm của toàn xe |

Hai dòng trên là hướng xử lý từ số đo hiện tại. Sau khi sửa trên CVAT, cần Save và chạy lại `sync` để có số đo sau sửa; không sửa trực tiếp hai CSV.

Phiếu tự kiểm hiện ghi nhận bốn cuboid có cảnh báo:

| Frame, idx | Nội dung cảnh báo từ phiếu tự kiểm | Số điểm trong hộp |
|---|---|---:|
| s02f42, 1 | Rộng khoảng 3,47 m, cao khoảng 0,86 m, ngoài khoảng kiểm tra của Car | 401 |
| s03f50, 6 | Cao 4,203 m, vượt cận kiểm tra 4,2 m của Bus | 135 |
| s05f05, 6 | Đáy cao hơn mặt đường 0,69 m; chiều cao khoảng 0,64 m, ngoài khoảng kiểm tra của Car | 718 |
| s05f31, 6 | Chiều cao khoảng 0,85 m, ngoài khoảng kiểm tra của Car | 608 |

**Nguyên tắc giữ cảnh báo:** không thay đổi hộp chỉ để bỏ cờ. Với Bus s03f50/6, chiều cao 4,203 m vượt cận 4,2 m chỉ 0,003 m; nếu đỉnh và đáy đã khớp vật thì có thể giữ kích thước và ghi lại mức chênh lệch này. Với các cảnh báo Car có chiều cao thấp hoặc khoảng hở lớn, ưu tiên kiểm lại toàn bộ chiều cao, mặt đường và vật lân cận trước khi quyết định. Việc giữ hay sửa từng hộp dựa vào hình học thực tế trên CVAT.

**Kiểm chéo**

MSSV người kiểm chéo: **2A202603290**.

Hai giá trị được chọn làm nội dung kiểm chéo:

| Số dùng để đối chiếu | Frame, idx | Giá trị và phép kiểm tra từ bảng A |
|---|---|---|
| Đáy cuboid | s01f13, 1 | z − h/2 = −1,006 − 1,392/2 = −1,702 m, bằng cột z_bottom |
| Khoảng cách ngang | s01f13, 2 | √(3,351² + (−18,236)²) ≈ 18,541 m, làm tròn thành 18,54 m, bằng cột dist_m |

Hai phép tính trên khớp với các cột tương ứng của bảng A ở mức làm tròn đang sử dụng. Kết quả trao đổi trực tiếp do MSSV 2A202603290 xác nhận; nếu có chênh lệch, kiểm lại frame, idx, đơn vị và phiên bản CSV trước khi cập nhật báo cáo.

## 6. Ý nghĩa của nhãn và hướng hoàn thiện

Mô hình học từ nhãn 3D cần vị trí và kích thước theo mét để biểu diễn vật trong không gian thực, phục vụ ước lượng khoảng cách và vùng chiếm chỗ. Với vật thưa điểm, quy tắc gán nhãn riêng giúp nhãn nhất quán và giảm việc suy diễn hình dạng vượt quá thông tin quan sát được.

Trọng tâm hoàn thiện là hướng đầu xe, đáy hộp và kích thước của bốn cuboid có cảnh báo. Các hộp ngoài 50 m cần được phân biệt với vùng chấm điểm của đề. Sau mỗi lần chỉnh cuboid và đồng bộ, các thống kê và số dẫn trong báo cáo cần khớp với bảng A và phiếu tự kiểm mới nhất.

Về dữ liệu, báo cáo chỉ dùng tên frame, class và số đo được phép công khai. Ảnh chụp giữ trong `report/individual/images/` và nộp bằng ZIP qua LMS; không đưa ảnh, point cloud hoặc thông tin nhận dạng lên repo public.
