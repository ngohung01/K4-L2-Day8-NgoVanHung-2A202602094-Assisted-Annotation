# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, nếu chỉ có ngân sách rà 5 ảnh, tôi ưu tiên các frame sau: `frame_0182.jpg`, `frame_0369.jpg`, `frame_0380.jpg`, `frame_0326.jpg`, và `frame_0331.jpg`. Chúng nằm ở top 5 về `score`, đều có điểm `U` cao và đều có `D = 1.0`, nghĩa là chúng là các cảnh có nhiều xe khó, có độ bất định cao và không quá lặp lại trong cùng một đoạn video. Đặc biệt, các frame này xuất hiện ở thời điểm rất khác nhau (từ 72.8s đến 150s+), nên nếu chọn cả 5 frame thì ta thu được đa dạng phong cảnh và đường đi tốt hơn so với chọn 5 cảnh liền nhau. 

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

- `frame_0182.jpg` — score 0.9591, `U = 0.9182`, `A = 1.0`, `D = 1.0`. Đây là cảnh có nhiều xe ở đường gần và khoảng cách độ sâu khác nhau; model có nguy cơ bỏ sót nhiều xe ở mép trái-phải. Đây là một ví dụ rất phù hợp để sửa label vì thêm box mới ở mép và sửa box gộp tạo ra thông tin có giá trị học tập cao.
- `frame_0369.jpg` — score 0.9324, `U = 0.9315`, `A = 0.8889`, `D = 1.0`. Cảnh này rất khó vì có sự chồng lấn, xe ở xa và xe gần cắt nhau. Đây là nơi model cho mỗi box gần đúng nhưng vẫn thiếu nhiều xe. Đây là một dự án cho dữ liệu active learning hiệu quả.
- `frame_0380.jpg` — score 0.9170, `U = 0.9340`, `A = 0.8333`, `D = 1.0`. Cảnh này có nhịp giao thông cao và nhiều xe ở cùng một hàng; mô hình thường gộp hoặc bỏ mất xe gần mép, do đó làm đầy thông tin label ở lô này sẽ cải thiện khả năng phát hiện xe ở vùng khó.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

- `frame_0372.jpg` có `score = 0.9101`, `U = 0.9202`, `A = 0.8333`, `D = 1.0`, nhưng tôi không chọn vì nó gần với `frame_0369.jpg` và `frame_0380.jpg` trong cùng phần đường và gần thời điểm tương tự. Nếu chọn quá nhiều cảnh cùng đoạn, các box mới rất có thể trùng lặp và không tăng thêm nhiều thông tin mới. Đây là ví dụ cho thấy `MIN_GAP_S` là cần thiết.
- `frame_0099.jpg` là một frame có ưu tiên khác và ngay cả khi không nằm trong top 5, nó vẫn đáng xem vì có nhiều box biến động theo hướng sửa `added`/`edited`; nếu ngân sách lớn hơn 5 ảnh, nó chắc chắn là ứng viên tiếp theo. 

Điều phép chọn này chưa chứng minh về chất lượng mô hình: nó chỉ chứng minh rằng các ảnh này có giá trị thông tin cao, không phải mô hình sẽ cải thiện ngay trên test set. Dù top frames có độ bất định cao, hiệu quả thực tế còn phụ thuộc vào chất lượng label, độ tương đồng giữa pool và test, cũng như định nghĩa “bản nhãn tham chiếu” ở tập test còn chỉ là bản tham chiếu mô hình, chứ chưa phải nhãn được con người rà tay. Vì vậy, lựa chọn này là tối ưu cho annotation value, không phải là xác nhận kết quả cuối cùng của mô hình.
