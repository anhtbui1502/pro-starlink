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
| 2 | Gói cước phụ (Add-On) | Tương tự offerType 7 (Family) / 8 (WhiteList-BlackList) của dịch vụ viễn thông | Hạn sử dụng là cuối chu kỳ hiện tại của gói chính. Lưu lượng cộng dồn, khi gia hạn gói chính thì reset | Mỗi thuê bao có thể có nhiều gói phụ, có thể đăng ký nhiều lần cùng 1 mã giá | Chỉ có action **đăng ký**, không có action gia hạn | Khi đăng ký: trừ X đồng từ tài khoản tiền, cộng dồn Y vào tài khoản lưu lượng |

**Kết luận:** Để đáp ứng chính sách của thuê bao Starlink, trên OCS chỉ cần khai báo **2 template** (1 template gói cước chính, 1 template gói cước phụ).

### 3. Phương pháp triển khai gói cước

Có 2 phương pháp triển khai gói cước trên OCS:

| Tiêu chí | Cách 1: Tham số hóa gói cước | Cách 2: Khai báo nhiều gói cước |
|---|---|---|
| Số lượng offer | Mỗi loại gói (gói chính, gói phụ) chỉ triển khai **1 offer** | Mỗi loại gói triển khai thành **nhiều offer** |
| Quản lý lưu lượng và số tiền trừ | Lưu lượng và số tiền trừ là các **tham số (param)**, tương tự dịch vụ Mysign | Lưu lượng và số tiền trừ do **OCS quản lý trong gói**, khai báo qua Product Catalog |
| Vai trò CNTT | CNTT truyền các tham số này khi đấu nối, đăng ký gói | CNTT chỉ truyền mã gói khi đấu nối, đăng ký gói |

**Lưu ý:** Mỗi cách tương ứng với một tập lệnh giao tiếp khác nhau ==> **cần chốt giải pháp** trước khi xây dựng tập lệnh Provisioning.
