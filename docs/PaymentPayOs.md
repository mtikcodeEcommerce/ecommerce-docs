# Hướng Dẫn Tích Hợp PayOS trong NestJS

## Giới thiệu

## 1. Setup Tài khoản PayOS
- Làm theo hướng dẫn sau: https://payos.vn/docs/huong-dan-su-dung/tao-tai-khoan-payos
- Kết nối tài khoản ngân hàng: https://payos.vn/docs/huong-dan-su-dung/ket-noi-tai-khoan-ngan-hang
- Tạo kênh thanh toán: https://payos.vn/docs/huong-dan-su-dung/kenh-thu/tao-kenh-thanh-toan
- Sau khi tạo xong kênh thanh toán sẽ có thông tin kết nối API
```.env
PAYOS_CLIENT_ID=
PAYOS_API_KEY=
PAYOS_CHECKSUM_KEY=
PAYOS_RETURN_URL=http://localhost:8080/api/v1/payments/return/payos
PAYOS_CANCEL_URL=http://localhost:8080/api/v1/payments/cancle/payos
```
