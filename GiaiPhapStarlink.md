# Giải pháp OCS đáp ứng dịch vụ Starlink

## I. Triển khai chính sách

### 1. Triển khai tài khoản

Trên OCS khai báo 3 loại tài khoản cho thuê bao Starlink:

| STT | Loại tài khoản | BalTypeId | Mục đích sử dụng |
|---|---|---|---|
| 1 | Tài khoản tiền | 1 | - Lưu cước đóng trước<br>- Gia hạn hàng tháng gói chính |
| 2 | Tài khoản lưu lượng | 11000 | - Lưu trữ lưu lượng hàng tháng của gói chính<br>- Cộng dồn lưu lượng các gói Add-On |
| 3 | Tài khoản Pool | 11001 | - Lưu trữ lưu lượng của gói Pool |

**Lưu ý:**

- VTT phải đồng nhất **1 đơn vị lưu lượng** khi truyền lên OCS.

### 2. Các tham số của gói cước chính

Gói cước chính được tham số hóa tương tự dịch vụ Mysign. **Tại 1 thời điểm, mỗi gói cước chính của thuê bao có 3 tham số** sau:

| STT | CharSpecId | Tên tham số | Ý nghĩa |
|---|---|---|---|
| 1 | 11001 | Loại gói cước | Lưu loại gói cước: LOCAL / GLOBAL |
| 2 | 11002 | Số tiền trừ 1 chu kỳ | Số tiền trừ từ tài khoản tiền (BalTypeId = 1) cho 1 chu kỳ (30 ngày hoặc 1 tháng) |
| 3 | 11003 | Lưu lượng 1 chu kỳ | Lưu lượng của 1 chu kỳ, được cộng reset vào tài khoản lưu lượng (BalTypeId = 11000) mỗi khi gia hạn |

### 3. Triển khai gói cước

**Lưu ý:** Gói cước **Add-On** và gói cước **Pool** **không quản lý trên OCS**. Đối với các gói cước này, sau khi kinh doanh thu tiền mặt của khách hàng, hệ thống chỉ gọi lên OCS để **cộng lưu lượng** vào tài khoản lưu lượng.

**OCS chỉ quản lý gói cước chính.** Gói cước chính được tổ chức thành **1 template** và **2 mã giá**:

<table>
<thead>
<tr><th>STT</th><th>Mã giá (ExternalId)</th><th>Chu kỳ</th><th>Action khai báo</th><th>Chính sách mỗi chu kỳ</th></tr>
</thead>
<tbody>
<tr><td>1</td><td>11000000</td><td>30 ngày</td><td rowspan="2">Đấu nối, Gia hạn</td><td rowspan="2">- Trừ tiền vào tài khoản tiền (BalTypeId = 1) 1 lượng = đúng giá trị tham số CharSpecId = 11002<br>- Cộng lưu lượng vào tài khoản lưu lượng (BalTypeId = 11000) 1 lượng = đúng giá trị tham số CharSpecId = 11003</td></tr>
<tr><td>2</td><td>11000001</td><td>1 tháng</td></tr>
</tbody>
</table>

## II. Tập lệnh giao tiếp Provisioning

### 0. Quy ước chung

**Endpoint:** OCS cung cấp tập lệnh qua 3 nhóm URL (tương tự dịch vụ Mysign):

| Nhóm lệnh | Method | URL |
|---|---|---|
| Lệnh nghiệp vụ (phân biệt bởi `processFlowSpecification`) | POST | `http://ipaddress:port/processFlowManagement/v4/processFlow/` |
| Nạp tiền | POST | `http://ipaddress:port/prepayBalanceManagement/v4/topupBalance` |
| Trừ lưu lượng / điều chỉnh tài khoản | POST | `http://ipaddress:port/prepayBalanceManagement/v4/balanceAdjust/` |

**Các trường dùng chung:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn / partyAccount.id | Định danh thuê bao Starlink trên OCS (thống nhất giữa Provisioning, BCCS và OCS) | Xâu ký tự, độ dài < 30 | Có |
| productExternalId | Mã giá gói cước khai báo trên OCS (Product Catalog) | Long | Theo lệnh |
| requestId | Mã giao dịch. Hệ thống client đảm bảo không trùng để phục vụ tra cứu, rollback | String | Theo lệnh |
| requestDate | Thời điểm thực hiện lệnh. OCS chỉ xử lý giao dịch có requestDate không quá 5 phút so với thời điểm OCS nhận bản tin | yyyy/MM/dd HH:mm:ss | Theo lệnh |
| channel | Kênh gửi lệnh. Đề xuất cố định = `STARLINK` (khai báo trên OCS) | String | Theo lệnh |
| partyCode / reason | Log giao dịch, OCS xuất ra CDR. Nếu không truyền mặc định `PROVISIONING_commandName` | String | Không |

**Output chung:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode / status | Mã lỗi (0: thành công, khác 0: thất bại) | Int |
| description | Mô tả lỗi | String |

**Mã lỗi chung** (áp dụng cho tất cả các lệnh, mã lỗi đặc thù được nêu trong từng lệnh):

| ErrorCode | Description | Mô tả |
|---|---|---|
| 0 | Success | Thành công |
| 1 | LACK PARAM IN FIELD + fieldName | Thiếu trường input bắt buộc |
| 1 | FORMAT INCORRECT IN FIELD + fieldName | Truyền input sai định dạng |
| 1 | WS_MSISDN_NOT_FOUND | Thuê bao không tồn tại trên OCS |
| 1 | WS_TRANS_DELAYED | requestDate quá 5 phút so với thời điểm OCS nhận bản tin |
| 1 | WS_NOT_CONFIG_CHANNEL_PREFIX | channel chưa được khai báo trên OCS |
| 1 | SUB_IN_BLOCK_ONE_WAY_BY_REQ_STATE_4407 | Thuê bao đang ở trạng thái chặn 1 chiều theo yêu cầu |
| 1 | WRONG_CONFIGURATION / ERROR_WHILE_IDENTIFING_CONFIGURATION | Lỗi khai báo chính sách gói cước trên OCS |
| 1 | WS_EXCEPTION_IN_POLICY_BLOCK | Ngoại lệ không mong muốn |
| 1 | WS_ABM_EXECUTE_ERROR | Lỗi cập nhật database OCS |

---

### 1. Đấu nối thuê bao Starlink

**Mô tả nghiệp vụ:**

- Provisioning gửi lệnh đấu nối thuê bao Starlink lên OCS với mã gói cước chính (Local/Global là các mã giá khác nhau) và số tiền cam kết đóng trước (CĐT).
- OCS tạo thuê bao ở trạng thái hoạt động, tạo tài khoản tiền (BalTypeId = 1) với số tiền = `initAmount`.
- OCS thực hiện ngay chu kỳ đầu tiên của gói chính: trừ X đồng (giá gói/chu kỳ, khai báo trong gói) từ tài khoản tiền, cộng Y (lưu lượng/chu kỳ) vào tài khoản lưu lượng (BalTypeId = 11000), hạn chu kỳ = thời điểm đấu nối + 30 ngày.
- Các chu kỳ tiếp theo OCS tự động gia hạn bằng tài khoản tiền (xem mục 11).
- Đề xuất: `initAmount` phải ≥ giá 1 chu kỳ của gói chính, nếu không OCS trả lỗi.
- Lệnh này cũng dùng để đấu nối thuê bao chủ nhóm pool (xem mục 12).

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "NewConnection",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "productExternalId", "value": "500001" },
    { "name": "imsi", "value": "KIT00123456" },
    { "name": "iccid", "value": "SL-SERVICE-LINE-0001" },
    { "name": "initAmount", "value": "12000000" },
    { "name": "subType", "value": "12" },
    { "name": "partyCode", "value": "Dau noi Starlink" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "NewConnection",
  "processFlowDate": "2026/11/01 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao Starlink | Xâu ký tự < 30 | Có |
| productExternalId | Mã gói cước chính (Local/Global, các mức lưu lượng) | Long | Có |
| imsi | Tham số bắt buộc trên OCS. Provisioning truyền mã Kit hoặc chuỗi định danh thiết bị. OCS xuất ra CDR | Xâu ký tự | Có |
| iccid | Tham số bắt buộc trên OCS. Provisioning truyền Service Line ID hoặc chuỗi định danh liên quan. OCS xuất ra CDR | Xâu ký tự | Có |
| initAmount | Số tiền CĐT (tổng số tiền đóng trước, đã gồm VAT) nạp vào tài khoản tiền | Long ≥ 0 | Có |
| subType | Loại thuê bao Starlink trên OCS. Giá trị do VTNET quy hoạch (ví dụ = 12) | Int | Có |
| partyCode | Log giao dịch | String | Không |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_MSISDN_EXIST | Thuê bao đã tồn tại |
| WS_OFFER_NOT_FOUND | Mã giá không tồn tại trên OCS |
| WS_WRONG_OFFER_TYPE | Mã giá truyền lên không phải gói chính |
| BALANCE_NOT_ENOUGH | initAmount không đủ trừ cước chu kỳ đầu tiên (đề xuất) |

---

### 2. Nạp tiền vào tài khoản thuê bao

**Mô tả nghiệp vụ:**

- Khách hàng nạp thêm tiền vào tài khoản tiền (BalTypeId = 1) để gia hạn gói chính các chu kỳ tiếp theo hoặc mua gói Add-On.
- OCS cộng tiền vào tài khoản tiền, lưu giao dịch theo `requestId` để phục vụ tra cứu và rollback (không quá 30 ngày).
- Lệnh nạp tiền chỉ cộng tiền. Trường hợp thuê bao đang ở trạng thái hết CĐT (gói chính chưa gia hạn được), Provisioning gọi lệnh Gia hạn gói chính (mục 5) sau khi nạp tiền.

**Request:** `POST .../prepayBalanceManagement/v4/topupBalance`

```json
{
  "requestId": "SL_TOPUP_0001",
  "requestDate": "2026/11/01 10:00:00",
  "channel": "STARLINK",
  "partyAccount": { "id": "SL0000123456" },
  "bucket": { "id": "1" },
  "voucher": "BANK_TXN_123456",
  "amount": "2000000",
  "reason": "Nap tien Starlink"
}
```

**Response:**

```json
{
  "status": "0",
  "description": "Success",
  "bucket": {
    "id": "1",
    "remainingValue": { "amount": "3000000" }
  }
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| requestId | Mã giao dịch nạp tiền, không trùng | String | Có |
| requestDate | Thời điểm thực hiện lệnh | yyyy/MM/dd HH:mm:ss | Có |
| channel | Kênh gửi lệnh, = STARLINK | String | Có |
| partyAccount.id | Định danh thuê bao | Xâu ký tự < 30 | Có |
| bucket.id | Mã tài khoản tiền, cố định = 1 | Long | Có |
| voucher | Mã giao dịch ngân hàng / mã SN | Xâu ký tự | Không |
| amount | Số tiền nạp (VNĐ) | Long > 0 | Có |
| reason | Log giao dịch | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| status | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| bucket.remainingValue.amount | Số dư tài khoản tiền sau khi nạp | Long |

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_BALTYPE_NOT_ALLOWED | bucket.id không phải tài khoản tiền |

---

### 3. Ghi nhận sử dụng và trừ lưu lượng

**Mô tả nghiệp vụ:**

- Provisioning tổng hợp lưu lượng sử dụng của thuê bao (từ hệ thống Starlink) và gửi lệnh trừ lưu lượng lên OCS theo đơn vị lưu lượng đã thống nhất.
- OCS trừ lưu lượng trên tài khoản lưu lượng (BalTypeId = 11000).
- Nếu lưu lượng còn lại nhỏ hơn lượng cần trừ: OCS trừ về 0 (không rollback vì lưu lượng đã thực tế sử dụng), trả kết quả thành công kèm số dư, đồng thời sinh event hết lưu lượng (mục 11) và CDR cảnh báo (mục 19).
- OCS lưu giao dịch theo `requestId` để phục vụ tra cứu và rollback.

**Request:** `POST .../prepayBalanceManagement/v4/balanceAdjust/`

```json
{
  "requestDate": "2026/11/01 10:00:00",
  "requestId": "SL_USAGE_0001",
  "channel": "STARLINK",
  "partyAccount": { "id": "SL0000123456" },
  "amount": 1024,
  "chargeType": 3,
  "usageStartTime": "2026/11/01 00:00:00",
  "usageEndTime": "2026/11/01 09:59:59",
  "adjustType": "DeductBalance",
  "reason": "Ghi nhan su dung"
}
```

**Response:**

```json
{
  "status": "0",
  "description": "Success",
  "bucket": {
    "id": "11000",
    "remainingValue": { "amount": "50176" }
  }
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| requestDate | Thời điểm thực hiện lệnh | yyyy/MM/dd HH:mm:ss | Có |
| requestId | Mã giao dịch, không trùng | String | Có |
| channel | Kênh gửi lệnh, = STARLINK | String | Có |
| partyAccount.id | Định danh thuê bao | Xâu ký tự < 30 | Có |
| amount | Lưu lượng sử dụng cần trừ (theo đơn vị lưu lượng thống nhất) | Long > 0 | Có |
| chargeType | Loại trừ cước. Đề xuất cố định = 3 (trừ lưu lượng Starlink) | Int | Có |
| usageStartTime | Thời điểm bắt đầu kỳ ghi nhận sử dụng. OCS xuất ra CDR | yyyy/MM/dd HH:mm:ss | Không |
| usageEndTime | Thời điểm kết thúc kỳ ghi nhận sử dụng. OCS xuất ra CDR | yyyy/MM/dd HH:mm:ss | Không |
| adjustType | Cố định = DeductBalance | String | Có |
| reason | Log giao dịch | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| status | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| bucket.remainingValue.amount | Lưu lượng còn lại sau khi trừ | Long |

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_SAME_REQUEST_ID | requestId đã tồn tại (gửi trùng giao dịch) |

---

### 4. Tra cứu thông tin tài khoản thuê bao

**Mô tả nghiệp vụ:**

- Provisioning/CSKH tra cứu thông tin thuê bao: trạng thái, số dư tài khoản tiền, lưu lượng còn lại, gói chính (hạn chu kỳ hiện tại) và danh sách gói Add-On đã đăng ký trong chu kỳ.
- Gói Add-On đăng ký nhiều lần cùng mã giá được trả về thành nhiều bản ghi.

**Request:** `POST .../processFlowManagement/v4/processFlow?fields=subInfo,balanceResponseList`

```json
{
  "processFlowSpecification": "QueryAccount",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "QueryAccount",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" },
    {
      "name": "subInfoResponse",
      "value": {
        "state": "3",
        "stateSet": "000000000000000",
        "balanceResponseList": [
          {
            "balType": "1",
            "balId": "54990701",
            "name": "Tai khoan tien",
            "value": "3000000",
            "effDate": "2026/11/01 10:00:00",
            "expDate": "2100/01/01 00:00:00"
          },
          {
            "balType": "11000",
            "balId": "54990702",
            "name": "Tai khoan luu luong",
            "value": "51200",
            "effDate": "2026/11/01 10:00:00",
            "expDate": "2026/12/01 10:00:00"
          }
        ],
        "mainProductInfoResponse": {
          "name": "STARLINK_LOCAL_1TB",
          "productExternalId": "500001",
          "effDate": "2026/11/01 10:00:00",
          "expDate": "2026/12/01 10:00:00"
        },
        "productResponseList": [
          {
            "name": "STARLINK_ADDON_100GB",
            "productExternalId": "600001",
            "effDate": "2026/11/10 08:00:00",
            "expDate": "2026/12/01 10:00:00",
            "state": "1"
          }
        ]
      }
    }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| state | Trạng thái thuê bao: 3 - hoạt động, 4 - chặn 1 chiều theo yêu cầu | Int |
| stateSet | Cờ trạng thái chi tiết của thuê bao | Xâu 15 ký tự |
| balanceResponseList | Danh sách tài khoản: balType (mã tài khoản), balId (id tài khoản trên OCS), name, value (số dư), effDate, expDate | List |
| mainProductInfoResponse | Gói chính: productExternalId, name, effDate (bắt đầu chu kỳ hiện tại), expDate (kết thúc chu kỳ hiện tại) | Object |
| productResponseList | Danh sách gói Add-On: productExternalId, name, effDate, expDate (= cuối chu kỳ gói chính), state | List |

---

### 5. Gia hạn gói cước chính

**Mô tả nghiệp vụ:**

- Thông thường OCS tự động gia hạn gói chính mỗi chu kỳ 30 ngày (mục 11).
- Lệnh này dùng khi thuê bao đang ở trạng thái hết CĐT (đến kỳ gia hạn nhưng tài khoản tiền không đủ) và khách hàng đã nạp thêm tiền: Provisioning gọi lệnh để gia hạn ngay.
- Xử lý: trừ X đồng từ tài khoản tiền, reset tài khoản lưu lượng = Y của gói chính (lưu lượng Add-On của chu kỳ cũ bị reset), chu kỳ mới = thời điểm gia hạn + 30 ngày. Sau khi gia hạn thành công OCS sinh event gia hạn thành công (mục 11) để Provisioning mở lại băng thông.
- Đề xuất: nếu gói chính đang trong chu kỳ hiệu lực thì OCS trả lỗi, không cho gia hạn chồng chu kỳ.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "Renew",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "productExternalId", "value": "500001" },
    { "name": "partyCode", "value": "Gia han goi chinh" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "Renew",
  "processFlowDate": "2026/12/05 09:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| productExternalId | Mã gói chính hiện tại của thuê bao | Long | Có |
| partyCode | Log giao dịch | String | Không |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_PRODUCT_NOT_EXIST | Thuê bao không có gói cước truyền lên |
| WS_PRODUCT_IS_ACTIVE | Gói chính đang trong chu kỳ hiệu lực (đề xuất) |
| BALANCE_NOT_ENOUGH | Tài khoản tiền không đủ để gia hạn |

---

### 6. Đổi gói cước chính

**Mô tả nghiệp vụ:**

- Khách hàng đổi sang gói chính khác (ví dụ Local ↔ Global, đổi mức lưu lượng).
- Xử lý (đề xuất):
  - Hủy gói chính cũ, gán gói chính mới.
  - Thực hiện ngay chu kỳ đầu của gói mới: trừ X đồng (giá gói mới) từ tài khoản tiền, reset tài khoản lưu lượng = Y của gói mới, chu kỳ mới = thời điểm đổi gói + 30 ngày.
  - Các gói Add-On của chu kỳ cũ bị hủy (do gắn với chu kỳ gói chính).
  - Tài khoản tiền được bảo lưu.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "ChangeMainProduct",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "productExternalId", "value": "500002" },
    { "name": "partyCode", "value": "Doi goi chinh" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "ChangeMainProduct",
  "processFlowDate": "2026/11/15 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| productExternalId | Mã gói chính mới | Long | Có |
| partyCode | Log giao dịch | String | Không |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_OFFER_NOT_FOUND | Mã gói chính mới không tồn tại trên OCS |
| WS_SAME_OFFER | Gói mới trùng gói chính hiện tại |
| WS_WRONG_OFFER_TYPE | Mã giá truyền lên không phải gói chính |
| BALANCE_NOT_ENOUGH | Tài khoản tiền không đủ để trừ cước gói mới |

---

### 7. Mua thêm gói Add-On (VAS)

**Mô tả nghiệp vụ:**

- Khách hàng mua thêm lưu lượng bằng gói Add-On. Cho phép đăng ký nhiều lần cùng 1 mã giá, kể cả khi đang có gói đó. Không có nghiệp vụ gia hạn.
- Điều kiện: gói chính đang trong chu kỳ hiệu lực (không ở trạng thái hết CĐT).
- Xử lý: trừ X đồng (giá gói Add-On) từ tài khoản tiền, cộng dồn Y vào tài khoản lưu lượng. Hạn sử dụng gói Add-On = cuối chu kỳ hiện tại của gói chính. Khi gói chính gia hạn sang chu kỳ mới thì lưu lượng được reset.
- OCS sinh CDR đăng ký gói phụ để Provisioning/CNTT đối soát.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "AddSupOffer",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "productExternalId", "value": "600001" },
    { "name": "partyCode", "value": "Mua goi Add-On" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "AddSupOffer",
  "processFlowDate": "2026/11/10 08:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| productExternalId | Mã gói Add-On | Long | Có |
| partyCode | Log giao dịch | String | Không |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_OFFER_NOT_FOUND | Mã gói Add-On không tồn tại trên OCS |
| WS_WRONG_OFFER_TYPE | Mã giá truyền lên là gói chính |
| WS_MAIN_PRODUCT_EXPIRED | Gói chính đang hết CĐT, không cho mua Add-On (đề xuất) |
| BALANCE_NOT_ENOUGH | Tài khoản tiền không đủ |

---

### 8. Hủy thuê bao Starlink

**Mô tả nghiệp vụ:**

- Khách hàng chấm dứt dịch vụ, Provisioning gửi lệnh hủy thuê bao.
- OCS chuyển thuê bao về trạng thái hủy. Nếu thuê bao đang là thành viên nhóm pool thì OCS xóa thuê bao khỏi nhóm.
- Sau x ngày (cấu hình theo gói chính) không có lệnh khôi phục, OCS tự động xóa profile thuê bao. Ở trạng thái hủy, OCS không xử lý giao dịch nào khác.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "Termination",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "Termination",
  "processFlowDate": "2027/01/01 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |

**Mô tả Output:** errorCode, description (theo output chung).

---

### 9. Chặn/mở thuê bao Starlink theo yêu cầu

**Mô tả nghiệp vụ:**

- Chặn: khách hàng tạm dừng dịch vụ, thuê bao chuyển về trạng thái chặn 1 chiều theo yêu cầu. OCS từ chối các lệnh tác động tài khoản (trừ lưu lượng, gia hạn, đổi gói, mua Add-On, lệnh nhóm), chỉ cho phép tra cứu, mở chặn và hủy thuê bao.
- Mở: thuê bao trở về trạng thái hoạt động bình thường.
- OCS sinh CDR thay đổi trạng thái (mục 19).

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "BlockOrUnblock",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "blockOrUnblockType", "value": "4" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "BlockOrUnblock",
  "processFlowDate": "2026/11/20 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| blockOrUnblockType | 4: chặn 1 chiều theo yêu cầu; 6: mở chặn 1 chiều theo yêu cầu | Int | Có |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| SUB_SUSPENDING_4505 | Chặn khi thuê bao đã ở trạng thái chặn 1 chiều |
| SUB_WRONG_STATE_4502 | Mở chặn khi thuê bao không ở trạng thái chặn 1 chiều |

---

### 10. Điều chỉnh giao dịch

Gồm 2 lệnh: rollback giao dịch theo requestId và điều chỉnh trực tiếp tài khoản.

#### 10.1. Rollback giao dịch nạp tiền / trừ lưu lượng

**Mô tả nghiệp vụ:**

- Rollback giao dịch nạp tiền (mục 2) hoặc trừ lưu lượng (mục 3) đã thành công, trong thời gian không quá 30 ngày, theo `originalRequestId`.
- Provisioning đảm bảo requestId giữa các lệnh nạp tiền và trừ lưu lượng là duy nhất.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "RollbackBalanceAction",
  "characteristic": [
    { "name": "requestDate", "value": "2026/11/02 10:00:00" },
    { "name": "requestId", "value": "SL_ROLLBACK_0001" },
    { "name": "originalRequestId", "value": "SL_TOPUP_0001" },
    { "name": "channel", "value": "STARLINK" },
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "partyCode", "value": "Rollback giao dich" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "RollbackBalanceAction",
  "processFlowDate": "2026/11/02 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| requestDate | Thời điểm thực hiện lệnh | yyyy/MM/dd HH:mm:ss | Có |
| requestId | Mã giao dịch rollback | String | Có |
| originalRequestId | Mã giao dịch gốc cần rollback | String | Có |
| channel | Kênh gửi lệnh, = STARLINK | String | Có |
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| partyCode | Log giao dịch | String | Không |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_SAME_REQUEST_ID | requestId trùng originalRequestId |
| WS_NOT_FOUND_TRANSACTION | Không tồn tại giao dịch gốc |
| WS_ORIGINAL_TRANS_NOT_SUCCESS | Giao dịch gốc không thành công |
| WS_TRANS_IS_ROLLBACKED | Giao dịch gốc đã được rollback |
| WS_NOT_PERMIT_ROLLBACK | Tài khoản không đủ để rollback giao dịch nạp tiền |

#### 10.2. Điều chỉnh tài khoản (ModBalance)

**Mô tả nghiệp vụ:**

- Điều chỉnh trực tiếp giá trị, hạn sử dụng của tài khoản tiền hoặc tài khoản lưu lượng (xử lý khiếu nại, ứng cứu thông tin).

**Request:** `POST .../prepayBalanceManagement/v4/balanceAdjust/`

```json
{
  "requestDate": "2026/11/02 10:00:00",
  "requestId": "SL_ADJ_0001",
  "channel": "STARLINK",
  "partyAccount": { "id": "SL0000123456" },
  "bucket": { "id": "11000" },
  "amount": 10240,
  "balance": "",
  "balId": "",
  "addDays": "",
  "expDate": "",
  "adjustType": "ModBalance",
  "reason": "Dieu chinh khieu nai"
}
```

**Response:**

```json
{
  "status": "0",
  "description": "Success"
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| requestDate | Thời điểm thực hiện lệnh | yyyy/MM/dd HH:mm:ss | Có |
| requestId | Mã giao dịch | String | Có |
| channel | Kênh gửi lệnh, = STARLINK | String | Có |
| partyAccount.id | Định danh thuê bao | Xâu ký tự < 30 | Có |
| bucket.id | Mã tài khoản cần điều chỉnh (1: tiền, 11000: lưu lượng, 11001: Pool) | Long | Có |
| balId | Id tài khoản trên OCS (phân biệt các tài khoản cùng balType) | Long | Không |
| amount | Lượng điều chỉnh (+ cộng, - trừ) | Long | Không |
| balance | Set số dư tài khoản | Long | Không |
| addDays | Số ngày gia hạn thêm cho tài khoản | Long | Không |
| expDate | Set hạn sử dụng tài khoản | yyyy/MM/dd HH:mm:ss | Không |
| adjustType | Cố định = ModBalance | String | Có |
| reason | Log giao dịch | String | Không |

**Mô tả Output:** status, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| BALANCE_NOT_ENOUGH | Số dư không đủ để trừ |

---

### 11. Job/Event nghiệp vụ hết lưu lượng hoặc hết CĐT

**Mô tả nghiệp vụ:** Đây là xử lý chủ động phía OCS (không phải lệnh Provisioning gọi vào):

- **Job gia hạn gói chính:** đến cuối mỗi chu kỳ 30 ngày, OCS tự động gia hạn:
  - Tài khoản tiền đủ: trừ X đồng, reset tài khoản lưu lượng = Y, sang chu kỳ mới. Sinh event `RENEW_SUCCESS`.
  - Tài khoản tiền không đủ (hết CĐT): không gia hạn, tài khoản lưu lượng về 0, gói chính chuyển trạng thái chờ gia hạn. Sinh event `OUT_OF_CDT` để Provisioning bóp băng thông. Khách hàng nạp tiền và Provisioning gọi lệnh Gia hạn (mục 5) để tiếp tục sử dụng.
- **Event hết lưu lượng:** khi tài khoản lưu lượng về 0 trong chu kỳ (mục 3), OCS sinh event `OUT_OF_DATA` để Provisioning bóp băng thông. Khi mua Add-On (mục 7) hoặc gia hạn thành công, OCS sinh event `DATA_AVAILABLE` / `RENEW_SUCCESS` để Provisioning mở lại băng thông.

**Bản tin OCS gửi Provisioning (đề xuất):** `POST {URL callback do Provisioning cung cấp}`

```json
{
  "eventId": "OCS_EVT_000001",
  "eventType": "OUT_OF_DATA",
  "eventTime": "2026/11/25 13:45:00",
  "msisdn": "SL0000123456",
  "productExternalId": "500001",
  "cycleExpDate": "2026/12/01 10:00:00",
  "moneyBalance": "0",
  "dataBalance": "0"
}
```

**Response Provisioning trả OCS:**

```json
{
  "errorCode": "0",
  "description": "Success"
}
```

**Mô tả các trường:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| eventId | Mã event do OCS sinh, không trùng | String | Có |
| eventType | Loại event: OUT_OF_DATA (hết lưu lượng), OUT_OF_CDT (hết CĐT, không gia hạn được), RENEW_SUCCESS (gia hạn thành công), DATA_AVAILABLE (có lưu lượng trở lại sau khi mua Add-On) | String | Có |
| eventTime | Thời điểm phát sinh event | yyyy/MM/dd HH:mm:ss | Có |
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| productExternalId | Mã gói chính hiện tại | Long | Có |
| cycleExpDate | Hạn chu kỳ hiện tại của gói chính | yyyy/MM/dd HH:mm:ss | Có |
| moneyBalance | Số dư tài khoản tiền | Long | Có |
| dataBalance | Lưu lượng còn lại | Long | Có |
| errorCode (response) | 0: Provisioning nhận thành công; khác 0: OCS gửi lại theo cấu hình retry | Int | Có |

---

### 12. Đăng ký gói pool Starlink

**Mô tả nghiệp vụ:**

- Khách hàng doanh nghiệp mua gói pool (tổng lưu lượng dùng chung cho nhiều thuê bao Starlink).
- Tương tự nhóm doanh nghiệp của Mysign: Provisioning đấu nối 1 **thuê bao chủ nhóm (thuê bao ảo)** với gói chính là gói pool, định danh đề xuất theo định dạng `SL_{mã nhóm/mã hợp đồng}`.
- OCS tạo thuê bao chủ nhóm, tài khoản tiền = `initAmount`, trừ giá gói pool và cộng tổng lưu lượng pool vào tài khoản lưu lượng của thuê bao chủ nhóm, theo chu kỳ 30 ngày như gói chính.
- Sau khi đăng ký gói pool thành công, Provisioning gửi lệnh Tạo nhóm (mục 13).

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "NewConnection",
  "characteristic": [
    { "name": "msisdn", "value": "SL_POOL_000001" },
    { "name": "productExternalId", "value": "700001" },
    { "name": "imsi", "value": "SL_POOL_000001" },
    { "name": "iccid", "value": "SL_POOL_000001" },
    { "name": "initAmount", "value": "200000000" },
    { "name": "subType", "value": "13" },
    { "name": "partyCode", "value": "Dang ky goi pool" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "NewConnection",
  "processFlowDate": "2026/11/01 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao chủ nhóm pool, định dạng `SL_{mã nhóm}` | Xâu ký tự < 30 | Có |
| productExternalId | Mã gói pool | Long | Có |
| imsi | Tham số bắt buộc trên OCS, truyền = msisdn | Xâu ký tự | Có |
| iccid | Tham số bắt buộc trên OCS, truyền = msisdn | Xâu ký tự | Có |
| initAmount | Số tiền CĐT của gói pool | Long ≥ 0 | Có |
| subType | Loại thuê bao chủ nhóm pool. Giá trị do VTNET quy hoạch (ví dụ = 13) | Int | Có |
| partyCode | Log giao dịch | String | Không |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:** như mục 1 (WS_MSISDN_EXIST, WS_OFFER_NOT_FOUND, WS_WRONG_OFFER_TYPE, BALANCE_NOT_ENOUGH).

---

### 13. Tạo nhóm thuê bao Starlink

**Mô tả nghiệp vụ:**

- Thuê bao chủ nhóm đã đăng ký gói pool tạo nhóm chia sẻ lưu lượng (block tổng lưu lượng).
- Cho phép thêm luôn thành viên khi tạo nhóm, mỗi thành viên được cấp hạn mức lưu lượng (`limit`). OCS kiểm tra tổng hạn mức các thành viên không vượt quá lưu lượng của nhóm.
- Id nhóm do OCS sinh và trả về trong response, Provisioning lưu lại để dùng cho các lệnh nhóm.
- Tập lệnh nhóm không kiểm tra trạng thái chặn của thuê bao thành viên; Provisioning kiểm tra trước khi gửi lệnh.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "CreateGroup",
  "characteristic": [
    { "name": "msisdn", "value": "SL_POOL_000001" },
    { "name": "groupType", "value": "5" },
    { "name": "productExternalId", "value": "700001" },
    { "name": "limit", "value": "10240000" },
    { "name": "maxMember", "value": "50" },
    { "name": "shareService", "value": "111111111" },
    { "name": "sharePolicy", "value": "2" },
    { "name": "shareType", "value": "1" },
    { "name": "shareTime", "value": { "shareTimeType": "0" } },
    {
      "name": "listMember",
      "value": [
        {
          "msisdn": "SL0000123456",
          "limit": "1024000",
          "shareService": "111111111",
          "shareTime": { "shareTimeType": "0" }
        }
      ]
    }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "CreateGroup",
  "processFlowDate": "2026/11/01 10:05:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" },
    { "name": "groupId", "value": "9100401" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Thuê bao chủ nhóm pool | Xâu ký tự < 30 | Có |
| groupType | Loại nhóm. Đề xuất = 5: block tổng lưu lượng | Long | Có |
| productExternalId | Mã gói pool | Long | Có |
| limit | Tổng lưu lượng của nhóm. Không truyền thì mặc định theo gói pool | Long | Không |
| maxMember | Số thành viên tối đa. Không truyền thì mặc định theo gói pool | Long | Không |
| shareService | Cố định = 111111111 | String | Có |
| sharePolicy | Cố định = 2 | Int | Có |
| shareType | Cố định = 1 | Int | Có |
| shareTime.shareTimeType | Cố định = 0 | Int | Có |
| listMember | Danh sách thành viên thêm vào nhóm | List | Không |
| listMember.msisdn | Thuê bao thành viên | Xâu ký tự < 30 | Có |
| listMember.limit | Hạn mức lưu lượng cấp cho thành viên | Long | Không |
| listMember.shareService | Cố định = 111111111 | String | Có |
| listMember.shareTime.shareTimeType | Cố định = 0 | Int | Có |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| groupId | Id nhóm do OCS sinh | Long |

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_MSISDN_NOT_FOUND:+msisdn | Thuê bao thành viên không tồn tại |
| WS_OFFER_NOT_EXIST | Thuê bao chủ nhóm không có gói pool |
| WS_LIMIT_MUST_BE_LESS_OR_EQUAL_GROUP_LIMIT | Tổng hạn mức thành viên vượt lưu lượng nhóm |
| OVER_MAX_MEMBER | Số thành viên vượt maxMember |

---

### 14. Tra cứu nhóm và thành viên Starlink

**Mô tả nghiệp vụ:**

- Lệnh mới (Mysign chưa có). Tra cứu thông tin nhóm pool: gói pool, tổng lưu lượng, lưu lượng còn lại chưa cấp, hạn chu kỳ, danh sách thành viên và hạn mức/lưu lượng còn lại của từng thành viên.
- Truyền `groupId` để tra cứu 1 nhóm; không truyền thì trả về tất cả nhóm của thuê bao chủ nhóm.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "QueryGroup",
  "characteristic": [
    { "name": "masterMsisdn", "value": "SL_POOL_000001" },
    { "name": "groupId", "value": "9100401" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "QueryGroup",
  "processFlowDate": "2026/11/15 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" },
    {
      "name": "groupList",
      "value": [
        {
          "groupId": "9100401",
          "groupType": "5",
          "productExternalId": "700001",
          "limit": "10240000",
          "remainingLimit": "9216000",
          "maxMember": "50",
          "memberCount": "1",
          "effDate": "2026/11/01 10:00:00",
          "expDate": "2026/12/01 10:00:00",
          "listMember": [
            {
              "msisdn": "SL0000123456",
              "limit": "1024000",
              "consumed": "204800",
              "remaining": "819200",
              "expDate": "2026/12/01 10:00:00"
            }
          ]
        }
      ]
    }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| masterMsisdn | Thuê bao chủ nhóm pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| groupList.groupId | Id nhóm | Long |
| groupList.groupType | Loại nhóm | Int |
| groupList.productExternalId | Mã gói pool | Long |
| groupList.limit | Tổng lưu lượng của nhóm | Long |
| groupList.remainingLimit | Lưu lượng của nhóm chưa cấp cho thành viên | Long |
| groupList.maxMember | Số thành viên tối đa | Long |
| groupList.memberCount | Số thành viên hiện tại | Long |
| groupList.effDate / expDate | Bắt đầu / kết thúc chu kỳ hiện tại của nhóm | yyyy/MM/dd HH:mm:ss |
| listMember.msisdn | Thuê bao thành viên | Xâu ký tự |
| listMember.limit | Hạn mức lưu lượng được cấp | Long |
| listMember.consumed | Lưu lượng đã sử dụng | Long |
| listMember.remaining | Lưu lượng còn lại | Long |
| listMember.expDate | Hạn sử dụng tài khoản chia sẻ của thành viên | yyyy/MM/dd HH:mm:ss |

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_GROUP_NOT_EXIST | Không tồn tại nhóm |

---

### 15. Thêm thành viên vào nhóm Starlink

**Mô tả nghiệp vụ:**

- Thêm 1 hoặc nhiều thuê bao Starlink vào nhóm pool đang hoạt động, mỗi thành viên được cấp hạn mức lưu lượng.
- OCS trừ hạn mức của thành viên vào lưu lượng còn lại chưa cấp của nhóm, báo lỗi nếu vượt quá hoặc vượt số thành viên tối đa.
- Hạn sử dụng tài khoản chia sẻ của thành viên = hạn chu kỳ của nhóm.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "AddGroupMember",
  "characteristic": [
    { "name": "msisdn", "value": "SL_POOL_000001" },
    { "name": "groupId", "value": "9100401" },
    {
      "name": "listMember",
      "value": [
        {
          "msisdn": "SL0000123457",
          "limit": "512000",
          "shareService": "111111111",
          "shareTime": { "shareTimeType": "0" }
        }
      ]
    }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "AddGroupMember",
  "processFlowDate": "2026/11/15 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Thuê bao chủ nhóm pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Có |
| listMember | Danh sách thành viên cần thêm | List | Có |
| listMember.msisdn | Thuê bao thành viên | Xâu ký tự < 30 | Có |
| listMember.limit | Hạn mức lưu lượng cấp cho thành viên | Long | Không |
| listMember.shareService | Cố định = 111111111 | String | Có |
| listMember.shareTime.shareTimeType | Cố định = 0 | Int | Có |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_MSISDN_NOT_FOUND:+msisdn | Thuê bao thành viên không tồn tại |
| WS_SHARE_GROUP_NOT_FOUND | Không tồn tại nhóm |
| WS_LIMIT_MUST_BE_LESS_OR_EQUAL_GROUP_LIMIT | Hạn mức vượt lưu lượng còn lại của nhóm |
| OVER_MAX_MEMBER | Số thành viên vượt maxMember |
| WS_NOT_PERMISSION | Thuê bao đã là thành viên nhóm |

---

### 16. Hủy thành viên khỏi nhóm Starlink

**Mô tả nghiệp vụ:**

- Loại 1 hoặc nhiều thành viên khỏi nhóm pool.
- OCS cộng lưu lượng còn lại của thành viên về lưu lượng chưa cấp của nhóm (nếu tài khoản chia sẻ của thành viên còn hạn), sau đó xóa tài khoản chia sẻ của thành viên.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "RemoveGroupMember",
  "characteristic": [
    { "name": "masterMsisdn", "value": "SL_POOL_000001" },
    { "name": "groupId", "value": "9100401" },
    {
      "name": "listMemberMsisdn",
      "value": [
        { "msisdn": "SL0000123457" }
      ]
    }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "RemoveGroupMember",
  "processFlowDate": "2026/11/20 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| masterMsisdn | Thuê bao chủ nhóm pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Có |
| listMemberMsisdn | Danh sách thuê bao thành viên cần hủy | List xâu ký tự | Có |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_MSISDN_NOT_FOUND:+msisdn | Thuê bao thành viên không tồn tại |
| WS_GROUP_NOT_EXIST | Không tồn tại nhóm |
| WS_NOT_FOUND_UPPER_SHARE+memberMsisdn | Thuê bao chưa là thành viên nhóm |

---

### 17. Cập nhật hạn mức thành viên nhóm Starlink

**Mô tả nghiệp vụ:**

- Thay đổi hạn mức lưu lượng cấp cho 1 hoặc nhiều thành viên.
- Provisioning truyền hạn mức mới sau khi thay đổi (ví dụ đang 100 GB, muốn đổi thành 150 GB thì truyền 150 GB theo đơn vị thống nhất).
- Tăng: OCS kiểm tra lưu lượng chưa cấp của nhóm có đủ không, nếu không trả lỗi. Giảm: OCS trừ hạn mức và lưu lượng còn lại của thành viên, cộng phần chênh lệch về lưu lượng chưa cấp của nhóm.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "UpdateGroup",
  "characteristic": [
    { "name": "masterMsisdn", "value": "SL_POOL_000001" },
    { "name": "groupId", "value": "9100401" },
    {
      "name": "listMember",
      "value": [
        { "msisdn": "SL0000123456", "limit": "1536000" }
      ]
    }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "UpdateGroup",
  "processFlowDate": "2026/11/20 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| masterMsisdn | Thuê bao chủ nhóm pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Có |
| listMember | Danh sách thành viên cần điều chỉnh | List | Có |
| listMember.msisdn | Thuê bao thành viên | Xâu ký tự < 30 | Có |
| listMember.limit | Hạn mức lưu lượng sau khi điều chỉnh | Long ≥ 0 | Có |

**Mô tả Output:** errorCode, description (theo output chung).

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_GROUP_NOT_EXIST | Không tồn tại nhóm |
| WS_NOT_FOUND_UPPER_SHARE+memberMsisdn | Thuê bao chưa là thành viên nhóm |
| WS_LIMIT_MUST_BE_LESS_OR_EQUAL_GROUP_LIMIT | Hạn mức tăng vượt lưu lượng chưa cấp của nhóm |

---

### 18. Tra cứu báo cáo sử dụng pool Starlink

**Mô tả nghiệp vụ:**

- Lệnh mới (Mysign chưa có). Tra cứu báo cáo sử dụng lưu lượng của nhóm pool trong chu kỳ hiện tại: tổng lưu lượng, đã cấp, đã sử dụng, còn lại của nhóm và chi tiết theo từng thành viên.
- Có thể lọc theo 1 thuê bao thành viên.
- Đề xuất: OCS chỉ trả số liệu chu kỳ hiện tại (realtime); báo cáo các chu kỳ trước được tổng hợp từ CDR phía CNTT.

**Request:** `POST .../processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "QueryGroupUsage",
  "characteristic": [
    { "name": "masterMsisdn", "value": "SL_POOL_000001" },
    { "name": "groupId", "value": "9100401" },
    { "name": "memberMsisdn", "value": "" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "QueryGroupUsage",
  "processFlowDate": "2026/11/20 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" },
    {
      "name": "groupUsage",
      "value": {
        "groupId": "9100401",
        "cycleEffDate": "2026/11/01 10:00:00",
        "cycleExpDate": "2026/12/01 10:00:00",
        "totalLimit": "10240000",
        "allocated": "1536000",
        "consumed": "204800",
        "remaining": "10035200",
        "memberUsageList": [
          {
            "msisdn": "SL0000123456",
            "limit": "1536000",
            "consumed": "204800",
            "remaining": "1331200"
          }
        ]
      }
    }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| masterMsisdn | Thuê bao chủ nhóm pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Có |
| memberMsisdn | Lọc theo 1 thuê bao thành viên. Không truyền thì trả tất cả thành viên | Xâu ký tự < 30 | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| groupUsage.groupId | Id nhóm | Long |
| groupUsage.cycleEffDate / cycleExpDate | Bắt đầu / kết thúc chu kỳ hiện tại | yyyy/MM/dd HH:mm:ss |
| groupUsage.totalLimit | Tổng lưu lượng của nhóm | Long |
| groupUsage.allocated | Tổng hạn mức đã cấp cho thành viên | Long |
| groupUsage.consumed | Tổng lưu lượng thành viên đã sử dụng | Long |
| groupUsage.remaining | Lưu lượng còn lại của nhóm (= totalLimit - consumed) | Long |
| memberUsageList.msisdn | Thuê bao thành viên | Xâu ký tự |
| memberUsageList.limit | Hạn mức được cấp | Long |
| memberUsageList.consumed | Lưu lượng đã sử dụng | Long |
| memberUsageList.remaining | Lưu lượng còn lại | Long |

**Mã lỗi đặc thù:**

| Description | Mô tả |
|---|---|
| WS_GROUP_NOT_EXIST | Không tồn tại nhóm |
| WS_NOT_FOUND_UPPER_SHARE+memberMsisdn | Thuê bao chưa là thành viên nhóm |

---

### 19. CDR cảnh báo dung lượng, thời hạn gói và trạng thái chặn thuê bao Starlink

**Mô tả nghiệp vụ:**

- OCS sinh CDR cảnh báo khi:
  - Lưu lượng còn lại đạt các ngưỡng cấu hình (ví dụ còn 20%, 0%).
  - Gói chính sắp hết chu kỳ (trước N ngày, cấu hình) và đã hết chu kỳ (gia hạn thành công / hết CĐT).
  - Thuê bao thay đổi trạng thái (chặn/mở theo yêu cầu, hủy).
- CNTT/Provisioning dùng CDR để gửi thông báo khách hàng và đối soát. Ngưỡng cảnh báo và N ngày do kinh doanh quy định, OCS khai báo cấu hình.

**Cấu trúc CDR (đề xuất):**

| STT | Tên trường | Ý nghĩa | Định dạng |
|---|---|---|---|
| 1 | MSISDN | Định danh thuê bao | Xâu ký tự |
| 2 | ALERT_TYPE | Loại cảnh báo: 1 - ngưỡng lưu lượng; 2 - thời hạn gói; 3 - trạng thái thuê bao | Int |
| 3 | ALERT_CODE | Mã chi tiết: DATA_THRESHOLD, OUT_OF_DATA, CYCLE_EXPIRING, RENEW_SUCCESS, OUT_OF_CDT, BLOCK, UNBLOCK, TERMINATE | String |
| 4 | ALERT_TIME | Thời điểm phát sinh | yyyyMMddHHmmss |
| 5 | PRICE_PLAN_ID | Mã gói chính | Long |
| 6 | THRESHOLD | Ngưỡng cảnh báo (%) – với ALERT_TYPE = 1 | Int |
| 7 | BAL_TYPE_ID | Mã tài khoản liên quan | Long |
| 8 | DATA_BALANCE | Lưu lượng còn lại | Long |
| 9 | MONEY_BALANCE | Số dư tài khoản tiền | Long |
| 10 | CYCLE_EXP_DATE | Hạn chu kỳ hiện tại của gói chính | yyyyMMddHHmmss |
| 11 | BEFORE_STATE_SET | Trạng thái trước thay đổi – với ALERT_TYPE = 3 | Xâu ký tự |
| 12 | NOW_STATE_SET | Trạng thái sau thay đổi – với ALERT_TYPE = 3 | Xâu ký tự |
| 13 | GROUP_ID | Id nhóm pool (nếu là thành viên/chủ nhóm) | Long |

---

### Các điểm cần thống nhất

1. Định danh thuê bao Starlink (`msisdn`), giá trị `channel`, `subType` (thuê bao thường / chủ nhóm pool), `chargeType` của lệnh trừ lưu lượng.
2. Đơn vị lưu lượng thống nhất (MB/GB) và tần suất Provisioning gửi lệnh ghi nhận sử dụng.
3. Điều kiện `initAmount` ≥ giá chu kỳ đầu khi đấu nối.
4. Sau khi nạp tiền ở trạng thái hết CĐT: Provisioning gọi lệnh Gia hạn (như đề xuất) hay OCS tự động gia hạn.
5. Đổi gói chính: hủy Add-On chu kỳ cũ, bắt đầu chu kỳ mới ngay, không hoàn tiền chu kỳ cũ.
6. Phương thức thông báo event (mục 11): OCS gọi callback sang Provisioning hay chỉ qua CDR.
7. Báo cáo sử dụng pool (mục 18): chỉ chu kỳ hiện tại hay theo khoảng thời gian.
8. Ngưỡng cảnh báo lưu lượng và số ngày cảnh báo trước hết chu kỳ (mục 19).
9. Các lệnh Mysign sẵn có chưa đưa vào PYC, có thể bổ sung nếu cần: Khôi phục thuê bao (`RestoreConnection`), Hủy gói Add-On (`ModifySupOffer`), Hủy nhóm (`RemoveGroup`).
