# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Đặng Văn Huy

Công cụ gán nhãn đã dùng: CVAT Docker trên máy cá nhân

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian với vùng đệm (buffer gap) ở giữa thay vì chia ngẫu nhiên xuất phát từ đặc tính vật lý của dữ liệu video giám sát giao thông. Video được quay từ camera cố định trên cao, một chiếc xe di chuyển trên cao tốc sẽ xuất hiện liên tục trong khung hình qua nhiều frame liền kề suốt vài giây. 

Nếu chia ngẫu nhiên (random split), các frame liền kề của cùng một chiếc xe sẽ bị phân tán vào cả tập học (train) và tập kiểm thử (test), gây ra hiện tượng rò rỉ dữ liệu (data leakage). Khi đó, mô hình chỉ cần "học vẹt" đặc trưng của một chiếc xe cụ thể ở tập train là có thể phát hiện đúng chiếc xe đó ở tập test. Điều này khiến số đo hiệu năng (AP50, Precision, Recall) bị thổi phồng giả tạo, lệch hẳn theo hướng quá lạc quan so với năng lực thực tế. Việc chia theo thời gian cùng khoảng đệm cách ly đảm bảo tập kiểm thử hoàn toàn là các xe mới và tình huống mới, phản ánh trung thực khả năng tổng quát hóa của bộ phát hiện xe.

## 2. Mô hình khởi đầu lạnh (cold start)

Bảng số đo vòng khởi đầu lạnh trích xuất từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Dựa vào `outputs/compare_round0.jpg` và các chỉ số trên:
- Mô hình khởi đầu lạnh (YOLOv8n pretrained trên COCO) đạt độ chính xác khá tốt (P=0.925) nhưng độ phủ rất thấp (R=0.489). 
- Sự chênh lệch độ phủ thể hiện rõ rệt theo kích thước xe: xe nhỏ ở xa có Recall chỉ đạt 0.182 (bỏ sót hơn 81% số xe nhỏ), trong khi xe kích thước vừa đạt 0.547 và xe lớn đạt 0.561. Mô hình đặc biệt bỏ sót các xe ở xa chỉ có đốm đèn mờ hoặc các xe tối màu bị chìm vào nền đường đêm.
- Một trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai: Nhãn tham chiếu của tập kiểm thử cũng được sinh tự động bằng một mô hình khác mà chưa có chuyên viên kiểm định thủ công 100%. Các vùng phản quang đèn pha xuống mặt đường ướt hoặc các cụm đèn pha xa bị nhãn tham chiếu bỏ sót nhưng mô hình nhận diện được (hoặc ngược lại) có thể bị tính phạt sai (FP/FN) một cách oan uổng. Cần có chuyên viên quan sát trực quan đối chiếu trước khi khẳng định mô hình dự đoán sai.

## 3. Chiến lược chọn mẫu

Công thức tính điểm ưu tiên chọn mẫu là `score = W_U·U + W_A·A + W_D·D` với các trọng số $W_U = 0.5$, $W_A = 0.3$, $W_D = 0.2$:
- `U` (Uncertainty - 50% trọng số): Thể hiện độ bất định trung bình của các dự đoán; mô hình càng phân vân về xác suất phân lớp thì điểm U càng cao.
- `A` (Ambiguity - 30% trọng số): Tỷ lệ các bounding box có độ tin cậy nằm trong vùng mơ hồ cần người xác nhận.
- `D` (Diversity - 20% trọng số): Độ phân kỳ và đa dạng về không gian đặc trưng hoặc khoảng cách thời gian.
- `MIN_GAP_S` (ngưỡng thời gian tối thiểu, quy định 2.0s): Đóng vai trò then chốt trong việc loại bỏ các ảnh gần như tĩnh. Vì camera quay cố định, hai frame cách nhau dưới 2 giây có góc nhìn và vị trí xe gần như trùng hệt nhau. Ngưỡng này buộc hệ thống phân tán các mẫu được chọn trải đều theo thời gian, tối ưu hóa ngân sách gán nhãn.

Cân nhắc giữa các frame từ `reports/SELECTION.md`:
- Ba frame được chọn: `frame_0182.jpg` (hạng 1, điểm 0.9591 - độ bất định và số box mơ hồ cao nhất), `frame_0099.jpg` (hạng 8, điểm 0.9063 - bỏ sót xe cận cảnh lớn), và `frame_0107.jpg` (hạng 14, điểm 0.8876 - cách frame_0099 3.2s, luồng xe dịch chuyển phức tạp).
- Một frame bị loại dù điểm rất cao: `frame_0372.jpg` (hạng 6, điểm 0.9101, cao hơn 7 frame được chọn) bị thuật toán bỏ qua vì thời điểm t=148.8s chỉ cách `frame_0369.jpg` (t=147.6s) 1.2 giây (< 2.0s). Bỏ qua ảnh này giúp tránh lãng phí công sức rà soát hai cảnh gần như tương đương.
- Điểm bất định **không chứng minh** ảnh đó sẽ cải thiện mô hình. Điểm bất định chỉ cho biết mô hình hiện tại đang gặp khó khăn khi nhận dạng ảnh đó; nếu ảnh chứa nhiều nhiễu ban đêm hoặc khi số lượng ảnh đưa vào học lại quá ít làm mô hình bị co cụm dự đoán, kết quả sau fine-tune hoàn toàn có thể không cải thiện hoặc thậm chí giảm sút.

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp kết quả các vòng từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 270 | 0.378 | -0.393 | 1.000 | 0.042 | 0.081 | 0.000 | 0.024 | 0.244 |

Chi tiết sửa nhãn Vòng 1 theo `outputs/round1_diff.md`:
- Trên 12 ảnh được chọn, mô hình đề xuất 169 box pre-label. Sau khi thực hiện kiểm tra và chỉnh sửa trên CVAT, tổng số box đạt chuẩn là 270 box (tỷ lệ chấp nhận đạt 78%).
- Số lượng cụ thể: 132 box giữ nguyên (accepted), 19 box chỉnh sửa lại ranh giới (edited), 18 box xóa bỏ do mô hình dự đoán nhầm vệt đèn/khung ảo (deleted), và 119 box vẽ bổ sung cho các xe bị mô hình bỏ sót (added).
- AP50 của Vòng 1 đạt 0.378, giảm -0.393 so với khởi đầu lạnh (0.771).
- Về nhóm xe: Mô hình sau fine-tune đạt Precision tuyệt đối 1.000 (không có bất kỳ dự đoán False Positive nào trên tập test), nhưng Recall bị sụt giảm mạnh xuống 0.042 (chỉ bắt được 17 box, bỏ sót 386 box tại ngưỡng conf 0.25). Nhóm xe lớn giữ được Recall 0.244, xe vừa đạt 0.024, và xe nhỏ giảm về 0.000.

Đối chiếu ba khía cạnh thực tế:
- *Quan sát độc lập (`BLIND_SCAN.md`)*: Bằng mắt thường trên `frame_0099.jpg`, tôi ghi nhận khoảng 18 xe, chú ý thấy xe ở góc dưới bên phải bị mép ảnh cắt mất và các xe rất xa ở chân cầu chỉ còn hai chấm đèn, đồng thời nhận diện vệt sáng trên mặt đường không phải là xe.
- *Lỗi pre-label đã sửa (`REVIEW_LOG.csv`)*: Khi mở pre-label trên CVAT, tôi đã bổ sung box cho chiếc xe SUV màu tối ở làn giữa cận cảnh mà AI bỏ sót hoàn toàn (`added`), thu gọn box cho xe sedan tối màu làn phải để loại bỏ vệt phản quang màu đỏ trên mặt đường (`edited`), xóa khung ảo ở chân cầu (`deleted`), và giữ nguyên khung chuẩn của xe sedan đỏ (`accepted`).
- *Kết quả sau fine-tune (`compare_round1.jpg`)*: Khi fine-tune với tập mẫu nhỏ (12 ảnh), mô hình trở nên cực kỳ thận trọng: nó loại bỏ hoàn toàn các dự đoán nhiễu (FP = 0) nhưng lại bị hụt ngưỡng kích hoạt đối với các xe ở cự ly xa hoặc xe có độ tương phản thấp.
- *Ca khó theo guideline*: Trường hợp xe sedan màu tối có đèn hậu đỏ chiếu vệt sáng kéo dài trên mặt đường bê tông. Đúng theo guideline, ta chỉ được vẽ box ôm sát thân xe, không được khoanh trùm vệt phản chiếu ánh sáng.

## 5. Kết luận và giới hạn

So sánh với khởi đầu lạnh, Vòng 1 cho thấy mô hình sau khi học trên 12 ảnh ban đêm đã tinh chỉnh độ chính xác đạt mức tuyệt đối (P=1.000), song AP50 giảm từ 0.771 xuống 0.378 do Recall suy giảm sâu trên ngưỡng đánh giá cố định 0.25. 

Tôi quyết định **tiếp tục** sang Vòng 2 (nếu còn tài nguyên huấn luyện) nhằm bổ sung thêm đa dạng mẫu và khắc phục hiện tượng co cụm phân phối do số lượng ảnh train quá nhỏ (12 ảnh). 

Đề xuất hai ca còn yếu hoặc bất định cho vòng tiếp theo:
1. *Ca xe ở cự ly xa chỉ hiển thị hai đốm đèn pha hoặc đèn hậu nhỏ*: Đây là nhóm xe bị bỏ sót nhiều nhất (Recall small = 0.000), cần thêm dữ liệu để mô hình phân biệt giữa đèn xe ở xa và đèn đường/biển báo.
2. *Ca xe bị mép ảnh cắt hoặc xe bị che khuất một phần*: Cần thêm các ảnh có xe xuất hiện ở rìa khung hình để mô hình học cách bắt các đặc trưng bộ phận.

Chi phí và rủi ro: Rà soát các xe nhỏ ở cự ly xa tốn rất nhiều công sức phóng to/thu nhỏ trên CVAT; đồng thời cần tuyệt đối giữ `min_gap_s >= 2.0s` để tránh đưa vào các ảnh trùng lặp gây tốn chi phí vô ích.

Các giới hạn ảnh hưởng đến kết luận:
- Tập kiểm thử chỉ có 20 ảnh (cỡ mẫu nhỏ, dễ gây dao động thống kê lớn).
- Luật chấm điểm bỏ qua các box dưới 16 pixel có thể khiến một số ca phát hiện xe xa không được tính điểm.
- Nhãn tham chiếu do mô hình tự động tạo ra chứ chưa được chuyên gia thẩm định 100%, do đó không nên xem đây là chân lý tuyệt đối.

Nếu AP50 giảm, trước khi huấn luyện thêm vòng tiếp theo, tôi sẽ kiểm tra lại tính nhất quán của các box đã sửa trong `labels/round1/`, kiểm tra phân phối confidence score của mô hình trên tập test, và cân nhắc điều chỉnh siêu tham số fine-tune (như giảm số epoch từ 50 xuống 20–30, hoặc đóng băng một phần backbone) để chống overfit.
