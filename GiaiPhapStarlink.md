# Giải pháp OCS đáp ứng dịch vụ Starlink

## I. Triển khai chính sách

### 1. Triển khai tài khoản

Trên OCS khai báo 2 loại tài khoản cho thuê bao Starlink:

| STT | Loại tài khoản | BalTypeId | Mục đích sử dụng |
|---|---|---|---|
| 1 | Tài khoản tiền | 1 | - Lưu cước đóng trước<br>- Gia hạn hàng tháng gói chính<br>- Nạp tiền để mua gói Add-On |
| 2 | Tài khoản lưu lượng | X | - Lưu trữ lưu lượng hàng tháng của gói chính<br>- Cộng dồn lưu lượng các gói Add-On |

**Lưu ý:**

- Giá trị BalTypeId = X của tài khoản lưu lượng sẽ được xác định khi triển khai.
- VTT phải đồng nhất **1 đơn vị lưu lượng** khi truyền lên OCS.
