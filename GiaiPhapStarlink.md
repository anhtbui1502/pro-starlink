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

### 2. Triển khai gói cước

Trên OCS khai báo 2 loại gói cước cho thuê bao Starlink:

| STT | Loại gói cước | offerType | Chu kỳ | Quy tắc | Action khai báo | Xử lý khi thực hiện action |
|---|---|---|---|---|---|---|
| 1 | Gói cước chính | 1 | 30 ngày | Mỗi thuê bao chỉ có 1 gói cước chính | Chỉ có action **gia hạn** | Mỗi chu kỳ gia hạn: trừ X đồng từ tài khoản tiền, cộng Y vào tài khoản lưu lượng |
| 2 | Gói cước phụ (Add-On) | Tương tự offerType 7 (Family) / 8 (WhiteList-BlackList) của dịch vụ viễn thông | Không có | Mỗi thuê bao có thể có nhiều gói phụ, có thể đăng ký nhiều lần cùng 1 mã giá | Chỉ có action **đăng ký**, không có action gia hạn | Khi đăng ký: trừ X đồng từ tài khoản tiền, cộng dồn Y vào tài khoản lưu lượng |

**Kết luận:** Để đáp ứng chính sách của thuê bao Starlink, trên OCS chỉ cần khai báo **2 template** (1 template gói cước chính, 1 template gói cước phụ).
