# Quét độc lập trước khi xem pre-label

Frame: frame_0099.jpg, là một ảnh trong `to_label/round1/images/train/`

Số xe nhìn thấy bằng mắt: 26

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:
1. Vị trí ở bên phải ảnh có một chiếc xe đuôi mờ, màu đen và nằm liền mép đường.
2. Vị trí bên trái góc trên có các xe chỉ hiện một phần thân và đèn, nên dễ bị AI bỏ sót hoặc gộp vào vùng sáng.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
