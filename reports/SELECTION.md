# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, nếu chỉ có ngân sách rà 5 ảnh, tôi sẽ ưu tiên chọn 5 frame sau:
1. `frame_0182.jpg` (hạng 1, điểm 0.9591, thời điểm 72.8s): Đứng đầu toàn bộ tập pool về điểm số bất định, có U=0.9182 và A=1.0 (toàn bộ box có độ tin cậy phân vân), mật độ xe dày đặc vào ban đêm.
2. `frame_0369.jpg` (hạng 2, điểm 0.9324, thời điểm 147.6s): Điểm số rất cao, nằm ở phân đoạn thời gian khác biệt (>74s so với frame_0182) giúp bao quát bối cảnh giao thông mới.
3. `frame_0380.jpg` (hạng 3, điểm 0.9170, thời điểm 152.0s): Cách frame_0369 hơn 4s, đảm bảo tính đa dạng và có điểm bất định U=0.9340 rất cao.
4. `frame_0326.jpg` (hạng 4, điểm 0.9155, thời điểm 130.4s): Thuộc cụm thời gian giữa, mật độ xe đông đúc với nhiều xe đan xen phức tạp.
5. `frame_0099.jpg` (hạng 8, điểm 0.9063, thời điểm 39.6s): Dù đứng hạng 8 (sau frame_0331 hạng 5 và frame_0312 hạng 7), tôi ưu tiên chọn frame_0099 thay vì frame_0331. Lý do: frame_0331 (t=132.4s) cách frame_0326 (t=130.4s) đúng 2.0s nên cảnh gần như trùng lặp; trong khi frame_0099 ở thời điểm đầu video (t=39.6s), có U rất cao (0.9460) và thực tế mô hình không dự đoán được box cho chiếc xe SUV to ngay làn giữa cận cảnh, mang lại giá trị học tập vượt trội cho mô hình.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:
1. `frame_0182.jpg` (hạng 1, điểm 0.9591, t=72.8s): Trong CSV có U=0.9182, A=1.0, D=1.0, 28 box đề xuất và 18 box ambiguous. Contact sheet thể hiện rõ cảnh đêm nhiều làn xe với nhiều xe bị bóng tối che khuất một phần.
2. `frame_0099.jpg` (hạng 8, điểm 0.9063, t=39.6s): Trong CSV có U=0.9460, A=0.7778, D=1.0, 29 box đề xuất và 14 box ambiguous. Contact sheet và ảnh gốc cho thấy mô hình bị bối rối nặng bởi chùm đèn pha ngược chiều và vệt phản quang đỏ của đuôi xe trên mặt đường ướt.
3. `frame_0107.jpg` (hạng 14, điểm 0.8876, t=42.8s): Trong CSV có U=0.8752, A=0.8333, D=1.0, 33 box đề xuất với 15 box ambiguous. Contact sheet cho thấy luồng giao thông tiếp nối frame_0099 nhưng vị trí xe thay đổi, tạo ra nhiều trường hợp che khuất phức tạp.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
- `frame_0372.jpg` có điểm số rất cao (hạng 6, điểm 0.9101, U=0.9202, A=0.8333) cao hơn nhiều frame được chọn như frame_0312, frame_0099, frame_0187,... nhưng bị loại (`selected=False`). Lý do: frame này ở thời điểm t=148.8s, chỉ cách `frame_0369.jpg` (t=147.6s) vỏn vẹn 1.2 giây (nhỏ hơn ngưỡng `MIN_GAP_S = 2.0s`). Do góc quay camera tĩnh, hai ảnh cách nhau 1.2s gần như là một khung cảnh trùng lặp; việc loại bỏ giúp tiết kiệm ngân sách rà nhãn mà vẫn duy trì tính đa dạng.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:
- Điểm bất định cao chỉ chứng minh mô hình đang phân vân hoặc chưa chắc chắn về các dự đoán trong ảnh đó, chứ hoàn toàn chưa chứng minh rằng việc gán nhãn bổ sung cho ảnh đó sẽ giúp mô hình cải thiện hiệu năng (AP50) trên tập kiểm thử độc lập. Khi dữ liệu bổ sung chỉ có số lượng nhỏ (12 ảnh) và chứa nhiều trường hợp nhiễu ban đêm, việc fine-tune thậm chí có thể khiến mô hình bị co cụm dự đoán hoặc giảm độ phủ (recall).
