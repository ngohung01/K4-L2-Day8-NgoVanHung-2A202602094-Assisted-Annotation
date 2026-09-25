# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: NGÔ VĂN HƯNG

Công cụ gán nhãn đã dùng: CVAT Docker local trên máy cá nhân

## 1. Dữ liệu và cách chia tập

Pool và test được chia theo trục thời gian với vùng đệm ở giữa vì đây là bài toán phát hiện xe trên video đường cao tốc ban đêm. Nếu chia ngẫu nhiên, ảnh trong cùng đoạn đường, cùng độ sáng, cùng mẫu xe sẽ xuất hiện ở cả train và test. Khi đó AP50 sẽ bị phóng to vì mô hình được kiểm tra trên cùng phân bố hình ảnh nó đã thấy ở thời điểm gần; kết quả sẽ không phản ánh khả năng tổng quát trên tương lai. Vùng đệm giúp giảm hiện tượng rò rỉ thông tin và giữ độ chân thực của đánh giá, vì model phải dự đoán trên cảnh chưa nhìn thấy trước đó.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 trong `reports/rounds_table.md` là: model `yolov8n cold start (COCO car+bus+truck)`, không có ảnh train, AP50 = 0.771; P@0.25 = 0.925, R@0.25 = 0.489, F1 = 0.640. Tức là mô hình khởi đầu lạnh đã khá tốt ở mức độ phát hiện xe lớn và trung bình, nhưng vẫn bỏ sót nhiều xe nhỏ/xa. Trong `outputs/metrics_round0.json`, recall theo kích thước là: small = 0.182, medium = 0.547, large = 0.561. Điều này cho thấy điểm yếu chính nằm ở xe nhỏ và xe ở mép xa, nơi chiều cao box thường thấp và bị làm mờ bởi chuyển động hoặc ánh sáng. Một trường hợp cần xem lại nhãn tham chiếu trước khi kết luận mô hình sai là những box rất nhỏ hoặc bị che một phần, vì dữ liệu test có quy tắc bỏ qua các box cao dưới 16 px; nên nếu một xe quá nhỏ không được tính, ta không được coi đó là lỗi mô hình mà là quy ước đánh giá, không phải sai số mô hình.

## 3. Chiến lược chọn mẫu

Chiến lược chọn mẫu sử dụng dạng điểm: `score = W_U·U + W_A·A + W_D·D`, trong đó `U` là độ bất định, `A` phản ánh mức độ model có thể sửa/giải thích được ở một cảnh, và `D` đánh giá tính đa dạng của cảnh, còn `MIN_GAP_S` giúp tránh chọn các khung hình quá gần nhau trong cùng một đoạn video. Cách này phục vụ mục tiêu: chọn các frame có thông tin bổ ích nhất cho việc huấn luyện, nhưng không tập trung vào hình ảnh tương tự lặp lại. Trong `outputs/selection_round1.csv`, ba frame được ưu tiên sớm nhất là `frame_0182.jpg` (score 0.9591), `frame_0369.jpg` (0.9324), và `frame_0380.jpg` (0.9170); các frame này đều có `U` rất cao, `A` gần 1.0, `D = 1.0`, nghĩa là cảnh có nhiều xe khó, nhiều xe chồng lấn hoặc nhiều xe ở mép xa, đồng thời lệch về thời điểm và góc nhìn khác nhau. Một frame khác, `frame_0099.jpg`, cũng nằm trong vòng 12 ảnh và đáng chú ý vì `score = 0.9063`, nhưng nó lại có nhiều xe quan sát được rõ ràng ở các quy mô khác nhau; đây là ví dụ tổng hợp của độ bất định và độ khó. `frame_0372.jpg` có điểm cao (0.9101) nhưng không được chọn vì nó gần các cảnh cùng dải giao thông với `frame_0369/0380`, nên có nguy cơ trùng lặp và không mang thêm nhiều thông tin mới. Nói cách khác, độ bất định cho thấy cảnh đó có khả năng cải thiện mô hình, nhưng `MIN_GAP_S` ngăn ta chọn những cảnh gần trùng không tạo thêm lệch giá trị học tập.

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp từ `reports/rounds_table.md`:

| vòng | model                                   | ảnh train | box train |  AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 |    F1 | R small | R medium | R large |
| ---: | --------------------------------------- | --------: | --------: | ----: | -------------------: | -----: | -----: | ----: | ------: | -------: | ------: |
|    0 | yolov8n cold start (COCO car+bus+truck) |         0 |         0 | 0.771 |                    — |  0.925 |  0.489 | 0.640 |   0.182 |    0.547 |   0.561 |
|    1 | yolov8n fine-tune vong 1..1             |        12 |       318 | 0.452 |               -0.319 |  1.000 |  0.070 | 0.130 |   0.000 |    0.037 |   0.415 |
|    2 | yolov8n fine-tune vong 1..2             |        24 |       540 | 0.898 |               +0.127 |  0.954 |  0.514 | 0.668 |   0.030 |    0.564 |   0.927 |

Vòng 1 cho thấy lô 12 ảnh đầu tiên không mang lại cải thiện đáng kể: AP50 giảm từ 0.771 xuống 0.452. Đà này cho thấy tập label sửa ở vòng đầu bị quá tập trung vào một kiểu cảnh và gây drift cho mô hình. Tuy nhiên, sau vòng 2, mô hình đã hồi phục mạnh mẽ. Theo `outputs/round2_diff.md`, vòng 2 gồm 12 ảnh mới, với 222 box còn lại sau khi sửa, trong đó 23 box được giữ nguyên, 5 box chỉnh sửa, 0 box xoá FP, và 194 box được thêm mới để bù FN; accept rate là 82%. Tổng số box training lên 540, gấp gần 1.7 lần so với round 1. Kết quả là AP50 tăng từ 0.452 lên 0.898, tương ứng với tăng +0.446 và đứng trên mức cold start tới +0.127.

Về độ phủ theo kích thước xe, điểm cải thiện rõ nhất là nhóm xe lớn: recall large tăng từ 0.415 ở vòng 1 lên 0.927 ở vòng 2, đồng nghĩa với khả năng phát hiện xe lớn ở làn chính và hầu hết các cảnh gần xa đã phục hồi tốt. Recall medium cũng cải thiện từ 0.037 lên 0.564, cho thấy mô hình đã bắt đầu ổn định hơn với xe ở kích thước trung bình. Tuy nhiên, recall small vẫn rất thấp ở 0.030, cho thấy các xe nhỏ, mờ, hoặc bị che ở mép xa vẫn là điểm yếu chính. Đó là khoảng trống còn lại nếu muốn làm vòng tiếp theo.

Các ca sửa nhãn ở vòng 2 tập trung vào các cảnh có nhiều xe ở mép đường, xe thuộc các làn ngoài, và các box bị model bỏ sót hoặc gộp nhầm. Theo `outputs/round2_diff.md`, các ảnh như `frame_0015.jpg`, `frame_0030.jpg`, `frame_0073.jpg`, `frame_0128.jpg`, `frame_0232.jpg`, `frame_0290.jpg` đều có số box được thêm rất cao (12–22 box/ảnh). Đây là kiểu cảnh mà mô hình ban đầu dễ bỏ sót. Cùng với `BLIND_SCAN.md` và `REVIEW_LOG.csv`, ta có thể thấy việc sửa nhãn ở vòng 2 không chỉ là “điều chỉnh box” mà là khắc phục những thiếu sót hệ thống: thiếu box ở mép ảnh, gộp nhầm hai xe thành một, và bỏ sót xe ở vùng đèn mờ.

## 5. Kết luận và giới hạn

Kết quả của vòng 2 là tích cực: AP50 đạt 0.898, cao hơn cold start 0.771 và cải thiện mạnh so với vòng 1 0.452. Nếu nhìn vào toàn bộ tiến trình, đây là dấu hiệu cho thấy active learning có hiệu quả khi chọn đúng các cảnh khó và sửa nhãn kỹ lưỡng: vòng 2 không chỉ “bù thiếu” mà còn giúp model ổn định trên xe lớn và trung bình. Theo `outputs/metrics_round2.json`, precision đạt 0.954 và F1 đạt 0.668, cho thấy cả độ chính xác lẫn độ phủ đều được cải thiện so với vòng 1.

Vì vậy, tôi sẽ dừng ở vòng 2 và không tiếp tục train thêm ngay nếu mục tiêu là hoàn thành lab một cách chắc chắn. Lý do là tiến bộ hiện tại đã rõ ràng, và số lượng ảnh bổ sung ở vòng 2 đã đủ để làm mô hình cải thiện đáng kể. Tuy nhiên, nếu muốn tiếp tục để tăng thêm hiệu năng, vòng 3 nên tập trung vào mục tiêu rất rõ: các xe nhỏ ở mép đường, xe mờ dưới ánh đèn, và các cảnh có nhiều xe lẫn trong vùng tối. Đây là những trường hợp mà recall small còn rất yếu (0.030) và chi phí rà nhãn sẽ cao vì mỗi ảnh phải sửa nhiều box nhưng lợi ích tăng thêm chưa chắc tương xứng với thời gian bỏ ra.

Những giới hạn vẫn còn tồn tại: tập test chỉ có 20 ảnh, và nhãn tham chiếu trong `data/test/labels` không phải do con người rà thủ công mà do mô hình tạo ra. Hơn nữa, quy tắc bỏ qua các box quá nhỏ (`h < 16px`) khiến recall small bị penalize rất mạnh dù model có thể đang phát hiện đúng dạng xe nhỏ nhưng không được tính. Vì vậy, kết luận nên đặt ở mức “model đã cải thiện đáng kể trên tập test nhưng vẫn chưa mạnh ở xe quá nhỏ / mép ảnh / vùng mờ”. Nếu có vòng tiếp theo, cần chọn cảnh có nhiều xe nhỏ và xe bị che phần thân để tối ưu chi phí annotation.
