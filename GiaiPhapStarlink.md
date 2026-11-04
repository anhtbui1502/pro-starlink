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

### 1. Đấu nối thuê bao Starlink

**Mô tả nghiệp vụ:**

- Provisioning gửi lệnh đấu nối thuê bao Starlink lên OCS với:
  - Mã giá gói chính: 11000000 (chu kỳ 30 ngày) hoặc 11000001 (chu kỳ 1 tháng).
  - 3 tham số của gói chính: 11001 (loại gói LOCAL/GLOBAL), 11002 (số tiền trừ 1 chu kỳ), 11003 (lưu lượng 1 chu kỳ).
  - Số tiền cam kết đóng trước (CĐT) `initAmount`.
- OCS tạo thuê bao ở trạng thái hoạt động, lưu 3 tham số của gói chính, tạo tài khoản tiền (BalTypeId = 1) với số tiền = `initAmount`.
- OCS thực hiện ngay chu kỳ đầu tiên của gói chính: trừ tài khoản tiền 1 lượng = tham số 11002, cộng tài khoản lưu lượng (BalTypeId = 11000) 1 lượng = tham số 11003, hạn chu kỳ = thời điểm đấu nối + 30 ngày (mã giá 11000000) hoặc + 1 tháng (mã giá 11000001).
- Các chu kỳ tiếp theo OCS tự động gia hạn theo tham số hiện tại của gói chính (xem mục 9).
- Đề xuất: `initAmount` phải ≥ tham số 11002, nếu không OCS trả lỗi.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "NewConnection",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "productExternalId", "value": "11000000" },
    { "name": "imsi", "value": "KIT00123456" },
    { "name": "iccid", "value": "SL-SERVICE-LINE-0001" },
    { "name": "initAmount", "value": "12000000" },
    { "name": "subType", "value": "12" },
    {
      "name": "characteristicList",
      "value": [
        { "charSpecId": "11001", "stringValue": "LOCAL" },
        { "charSpecId": "11002", "longValue": "2000000" },
        { "charSpecId": "11003", "longValue": "1048576" }
      ]
    },
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
| msisdn | Định danh thuê bao Starlink (thống nhất giữa Provisioning, BCCS và OCS) | Xâu ký tự < 30 | Có |
| productExternalId | Mã giá gói chính: 11000000 (chu kỳ 30 ngày), 11000001 (chu kỳ 1 tháng) | Long | Có |
| imsi | Tham số bắt buộc trên OCS. Provisioning truyền mã Kit hoặc chuỗi định danh thiết bị. OCS xuất ra CDR | Xâu ký tự | Có |
| iccid | Tham số bắt buộc trên OCS. Provisioning truyền Service Line ID hoặc chuỗi định danh liên quan. OCS xuất ra CDR | Xâu ký tự | Có |
| initAmount | Số tiền CĐT (tổng số tiền đóng trước, đã gồm VAT) nạp vào tài khoản tiền | Long ≥ 0 | Có |
| subType | Loại thuê bao Starlink trên OCS. Giá trị do VTNET quy hoạch (ví dụ = 12) | Int | Có |
| characteristicList | Danh sách tham số gói chính, gồm đủ 3 tham số 11001, 11002, 11003 | List | Có |
| characteristicList.charSpecId | Id tham số: 11001 - loại gói; 11002 - số tiền trừ 1 chu kỳ; 11003 - lưu lượng 1 chu kỳ | String | Có |
| characteristicList.stringValue | Giá trị tham số 11001: LOCAL / GLOBAL | String | Có (với 11001) |
| characteristicList.longValue | Giá trị tham số 11002 (VNĐ) / 11003 (theo đơn vị lưu lượng thống nhất) | Long ≥ 0 | Có (với 11002, 11003) |
| partyCode | Log giao dịch. OCS xuất ra CDR. Nếu không truyền mặc định `PROVISIONING_commandName` | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_MSISDN_EXIST | Thuê bao đã tồn tại |
| WS_OFFER_NOT_FOUND | Mã giá không tồn tại trên OCS |
| WS_WRONG_OFFER_TYPE | Mã giá truyền lên không phải gói chính |
| WS_CHARACTERISTIC_INVALID | Thiếu hoặc sai giá trị tham số 11001 / 11002 / 11003 |
| BALANCE_NOT_ENOUGH | initAmount không đủ trừ cước chu kỳ đầu tiên (đề xuất) |

---

### 2. Nạp tiền vào tài khoản thuê bao

**Mô tả nghiệp vụ:**

- Khách hàng nạp thêm tiền vào tài khoản tiền (BalTypeId = 1) để gia hạn gói chính các chu kỳ tiếp theo.
- OCS cộng tiền vào tài khoản tiền, lưu giao dịch theo `requestId` để phục vụ tra cứu và rollback (không quá 30 ngày).
- Trường hợp thuê bao đang ở trạng thái hết CĐT (gói chính chưa gia hạn được), sự kiện nạp tiền trên OCS sẽ tự động gia hạn gói chính nếu tài khoản tiền sau khi nạp đủ (≥ tham số 11002): trừ tài khoản tiền = tham số 11002, reset tài khoản 11000 = tham số 11003, chu kỳ mới bắt đầu từ thời điểm nạp tiền. OCS sinh event `RENEW_SUCCESS` (mục 9) để Provisioning mở lại băng thông.

**Request:** `POST http://ipaddress:port/prepayBalanceManagement/v4/topupBalance`

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
| requestId | Mã giao dịch nạp tiền. Hệ thống client đảm bảo không trùng để phục vụ tra cứu, rollback | String | Có |
| requestDate | Thời điểm thực hiện lệnh. OCS chỉ xử lý giao dịch không quá 5 phút so với thời điểm OCS nhận bản tin | yyyy/MM/dd HH:mm:ss | Có |
| channel | Kênh gửi lệnh, cố định = STARLINK (khai báo trên OCS) | String | Có |
| partyAccount.id | Định danh thuê bao | Xâu ký tự < 30 | Có |
| bucket.id | Mã tài khoản tiền, cố định = 1 | Long | Có |
| voucher | Mã giao dịch ngân hàng / mã SN | Xâu ký tự | Không |
| amount | Số tiền nạp (VNĐ) | Long > 0 | Có |
| reason | Log giao dịch. OCS xuất ra CDR | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| status | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| bucket.remainingValue.amount | Số dư tài khoản tiền sau khi nạp | Long |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_BALTYPE_NOT_ALLOWED | bucket.id không phải tài khoản tiền |

---

### 3. Ghi nhận sử dụng và trừ lưu lượng

**Mô tả nghiệp vụ:**

- Provisioning tổng hợp lưu lượng sử dụng của thuê bao (từ hệ thống Starlink) và gửi lệnh trừ lưu lượng lên OCS theo đơn vị lưu lượng đã thống nhất.
- Thứ tự trừ (đề xuất):
  1. Tài khoản lưu lượng (BalTypeId = 11000): gồm lưu lượng chu kỳ gói chính và lưu lượng Add-On đã cộng dồn.
  2. Nếu thuê bao là thành viên nhóm Pool: trừ tiếp vào tài khoản chia sẻ Pool (BalTypeId = 11001) trong hạn mức được cấp.
- Nếu tổng lưu lượng còn lại nhỏ hơn lượng cần trừ: OCS trừ về 0 (không rollback vì lưu lượng đã thực tế sử dụng), trả kết quả thành công kèm số dư, đồng thời sinh event hết lưu lượng (mục 9) và CDR cảnh báo (mục 17).
- OCS lưu giao dịch theo `requestId` để phục vụ tra cứu và rollback.

**Request:** `POST http://ipaddress:port/prepayBalanceManagement/v4/balanceAdjust/`

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
    "remainingValue": { "amount": "1047552" }
  }
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| requestDate | Thời điểm thực hiện lệnh (không quá 5 phút so với thời điểm OCS nhận bản tin) | yyyy/MM/dd HH:mm:ss | Có |
| requestId | Mã giao dịch, không trùng | String | Có |
| channel | Kênh gửi lệnh, cố định = STARLINK | String | Có |
| partyAccount.id | Định danh thuê bao | Xâu ký tự < 30 | Có |
| amount | Lưu lượng sử dụng cần trừ (theo đơn vị lưu lượng thống nhất) | Long > 0 | Có |
| chargeType | Loại trừ cước. Đề xuất cố định = 3 (trừ lưu lượng Starlink) | Int | Có |
| usageStartTime | Thời điểm bắt đầu kỳ ghi nhận sử dụng. OCS xuất ra CDR | yyyy/MM/dd HH:mm:ss | Không |
| usageEndTime | Thời điểm kết thúc kỳ ghi nhận sử dụng. OCS xuất ra CDR | yyyy/MM/dd HH:mm:ss | Không |
| adjustType | Cố định = DeductBalance | String | Có |
| reason | Log giao dịch. OCS xuất ra CDR | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| status | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| bucket.remainingValue.amount | Lưu lượng còn lại trên tài khoản 11000 sau khi trừ | Long |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_SAME_REQUEST_ID | requestId đã tồn tại (gửi trùng giao dịch) |

---

### 4. Tra cứu thông tin tài khoản thuê bao

**Mô tả nghiệp vụ:**

- Provisioning/CSKH tra cứu thông tin thuê bao: trạng thái, số dư tài khoản tiền, lưu lượng còn lại (tài khoản 11000, tài khoản chia sẻ Pool 11001 nếu là thành viên nhóm), thông tin gói chính và hạn chu kỳ hiện tại.
- Tham số gói chính (11001, 11002, 11003) tra cứu bằng lệnh tại mục 5.3.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow?fields=subInfo,balanceResponseList`

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
            "value": "1047552",
            "effDate": "2026/11/01 10:00:00",
            "expDate": "2026/12/01 10:00:00"
          },
          {
            "balType": "11001",
            "balId": "54990703",
            "name": "Tai khoan Pool",
            "value": "512000",
            "effDate": "2026/11/05 10:00:00",
            "expDate": "2027/11/05 10:00:00",
            "groupId": "9100401"
          }
        ],
        "mainProductInfoResponse": {
          "name": "STARLINK_30_NGAY",
          "productExternalId": "11000000",
          "effDate": "2026/11/01 10:00:00",
          "expDate": "2026/12/01 10:00:00"
        }
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
| balanceResponseList | Danh sách tài khoản của thuê bao | List |
| balanceResponseList.balType | Mã tài khoản: 1 - tiền; 11000 - lưu lượng; 11001 - Pool | Long |
| balanceResponseList.balId | Id tài khoản trên OCS | Long |
| balanceResponseList.value | Số dư hiện tại | Long |
| balanceResponseList.effDate / expDate | Ngày hiệu lực / hết hạn của tài khoản | yyyy/MM/dd HH:mm:ss |
| balanceResponseList.groupId | Id nhóm Pool (chỉ có với tài khoản chia sẻ Pool) | Long |
| mainProductInfoResponse.productExternalId | Mã giá gói chính (11000000 / 11000001) | Long |
| mainProductInfoResponse.name | Tên gói chính | String |
| mainProductInfoResponse.effDate / expDate | Bắt đầu / kết thúc chu kỳ hiện tại của gói chính | yyyy/MM/dd HH:mm:ss |

**Mã lỗi:** Mã lỗi chung (mục 18).

---

### 5. Đổi gói cước chính

Với giải pháp tham số hóa, nghiệp vụ đổi gói chính gồm 2 trường hợp:

- Đổi loại gói (LOCAL/GLOBAL), giá hoặc lưu lượng: **thay đổi tham số** gói chính (mục 5.1).
- Đổi chu kỳ (30 ngày ↔ 1 tháng): **đổi mã giá** gói chính (mục 5.2).

#### 5.1. Thay đổi tham số gói cước chính

**Mô tả nghiệp vụ:**

- Thay đổi 1 hoặc nhiều tham số 11001, 11002, 11003 của gói chính.
- Tham số mới được áp dụng từ chu kỳ gia hạn tiếp theo.

**Request:** `PATCH http://ipaddress:port/accountManagement/v4/account/{msisdn}`

```json
{
  "requestDate": "2026/11/15 10:00:00",
  "channel": "STARLINK",
  "operations": [
    {
      "op": "addOrReplace",
      "path": "/characteristic?characteristic.charSpecId=11001",
      "value": { "stringValue": "GLOBAL" }
    },
    {
      "op": "addOrReplace",
      "path": "/characteristic?characteristic.charSpecId=11002",
      "value": { "longValue": 5000000 }
    },
    {
      "op": "addOrReplace",
      "path": "/characteristic?characteristic.charSpecId=11003",
      "value": { "longValue": 2097152 }
    }
  ]
}
```

**Response:**

```json
{
  "errorCode": "0",
  "description": "Success"
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao, truyền trên URL | Xâu ký tự < 30 | Có |
| requestDate | Thời điểm thực hiện lệnh (không quá 5 phút so với thời điểm OCS nhận bản tin) | yyyy/MM/dd HH:mm:ss | Có |
| channel | Kênh gửi lệnh, cố định = STARLINK | String | Có |
| operations | Danh sách tham số cần thay đổi | List | Có |
| operations.op | Cố định = addOrReplace | String | Có |
| operations.path | Tham số cần thay đổi: `/characteristic?characteristic.charSpecId={11001 / 11002 / 11003}` | String | Có |
| operations.value.stringValue | Giá trị mới của tham số 11001: LOCAL / GLOBAL | String | Có (với 11001) |
| operations.value.longValue | Giá trị mới của tham số 11002 (VNĐ) / 11003 (đơn vị lưu lượng thống nhất) | Long ≥ 0 | Có (với 11002, 11003) |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_CHARACTERISTIC_INVALID | charSpecId không thuộc gói chính hoặc giá trị không hợp lệ |

#### 5.2. Đổi mã giá gói cước chính (đổi chu kỳ)

**Mô tả nghiệp vụ:**

- Đổi giữa 2 mã giá 11000000 (chu kỳ 30 ngày) và 11000001 (chu kỳ 1 tháng).
- Có thể truyền kèm tham số mới; không truyền thì OCS giữ nguyên tham số hiện tại.
- Xử lý (đề xuất): hủy mã giá cũ, gán mã giá mới, bảo lưu tài khoản tiền và tài khoản Pool; bắt đầu ngay chu kỳ mới theo mã giá mới: trừ tài khoản tiền = tham số 11002, reset tài khoản 11000 = tham số 11003.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "ChangeMainProduct",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "productExternalId", "value": "11000001" },
    {
      "name": "characteristicList",
      "value": [
        { "charSpecId": "11001", "stringValue": "LOCAL" },
        { "charSpecId": "11002", "longValue": "2000000" },
        { "charSpecId": "11003", "longValue": "1048576" }
      ]
    },
    { "name": "partyCode", "value": "Doi chu ky goi chinh" },
    { "name": "reserveBalance", "value": "Y" }
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
| productExternalId | Mã giá gói chính mới (11000000 / 11000001) | Long | Có |
| characteristicList | Tham số gói chính mới (11001, 11002, 11003). Không truyền thì giữ nguyên tham số hiện tại | List | Không |
| partyCode | Log giao dịch. OCS xuất ra CDR | String | Không |
| reserveBalance | Bảo lưu tài khoản: Y - bảo lưu toàn bộ; B - bảo lưu các tài khoản khai báo cấu hình trên OCS | String | Có |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_OFFER_NOT_FOUND | Mã giá mới không tồn tại trên OCS |
| WS_SAME_OFFER | Mã giá mới trùng mã giá hiện tại |
| WS_WRONG_OFFER_TYPE | Mã giá truyền lên không phải gói chính |
| WS_CHARACTERISTIC_INVALID | Giá trị tham số không hợp lệ |
| BALANCE_NOT_ENOUGH | Tài khoản tiền không đủ để trừ cước chu kỳ mới |

#### 5.3. Truy vấn tham số gói cước chính

**Mô tả nghiệp vụ:** Tra cứu giá trị hiện tại của 3 tham số gói chính (11001, 11002, 11003) của thuê bao.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "UpdateListParameter",
  "characteristic": [
    { "name": "msisdn", "value": "SL0000123456" },
    { "name": "type", "value": "0" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "UpdateListParameter",
  "processFlowDate": "2026/11/15 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" },
    {
      "name": "characteristicList",
      "value": [
        { "charSpecId": "11001", "stringValue": "LOCAL", "state": "1" },
        { "charSpecId": "11002", "longValue": "2000000", "state": "1" },
        { "charSpecId": "11003", "longValue": "1048576", "state": "1" }
      ]
    }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| type | Cố định = 0 | Int | Có |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| characteristicList.charSpecId | Id tham số (11001, 11002, 11003) | String |
| characteristicList.stringValue / longValue | Giá trị tham số | String / Long |
| characteristicList.state | Trạng thái tham số (1 - hiệu lực) | Int |

**Mã lỗi:** Mã lỗi chung (mục 18).

---

### 6. Mua thêm gói Add-On (cộng lưu lượng Add-On)

**Mô tả nghiệp vụ:**

- Gói Add-On **không quản lý trên OCS**. Sau khi kinh doanh thu tiền mặt của khách hàng, Provisioning gọi lệnh lên OCS để **cộng lưu lượng** của gói Add-On vào tài khoản lưu lượng (BalTypeId = 11000).
- Cho phép gọi nhiều lần trong chu kỳ, lưu lượng được cộng dồn. Hạn sử dụng giữ nguyên = cuối chu kỳ hiện tại của gói chính; khi gói chính gia hạn sang chu kỳ mới thì lưu lượng được reset.
- Đề xuất: chỉ cho phép cộng khi gói chính đang trong chu kỳ hiệu lực (không ở trạng thái hết CĐT).
- OCS lưu giao dịch theo `requestId` để tra cứu và rollback (mục 8.1). Sau khi cộng thành công, nếu thuê bao đang bị bóp băng thông do hết lưu lượng, OCS sinh event `DATA_AVAILABLE` (mục 9).

**Request:** `POST http://ipaddress:port/prepayBalanceManagement/v4/balanceAdjust/`

```json
{
  "requestDate": "2026/11/10 08:00:00",
  "requestId": "SL_ADDON_0001",
  "channel": "STARLINK",
  "partyAccount": { "id": "SL0000123456" },
  "bucket": { "id": "11000" },
  "amount": 102400,
  "adjustType": "ModBalance",
  "reason": "Cong luu luong Add-On 100GB"
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
| requestDate | Thời điểm thực hiện lệnh (không quá 5 phút so với thời điểm OCS nhận bản tin) | yyyy/MM/dd HH:mm:ss | Có |
| requestId | Mã giao dịch, không trùng | String | Có |
| channel | Kênh gửi lệnh, cố định = STARLINK | String | Có |
| partyAccount.id | Định danh thuê bao | Xâu ký tự < 30 | Có |
| bucket.id | Mã tài khoản lưu lượng, cố định = 11000 | Long | Có |
| amount | Lưu lượng Add-On cần cộng (theo đơn vị lưu lượng thống nhất) | Long > 0 | Có |
| adjustType | Cố định = ModBalance | String | Có |
| reason | Log giao dịch (mã gói Add-On, mô tả). OCS xuất ra CDR | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| status | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_BALTYPE_NOT_ALLOWED | bucket.id không phải tài khoản lưu lượng |
| WS_MAIN_PRODUCT_EXPIRED | Gói chính đang hết CĐT, không cho cộng lưu lượng Add-On (đề xuất) |

---

### 7. Hủy thuê bao Starlink

**Mô tả nghiệp vụ:**

- Khách hàng chấm dứt dịch vụ, Provisioning gửi lệnh hủy thuê bao.
- OCS chuyển thuê bao về trạng thái hủy. Nếu thuê bao đang là thành viên nhóm Pool thì OCS xóa thuê bao khỏi nhóm.
- Sau x ngày (cấu hình theo gói chính) không có lệnh khôi phục, OCS tự động xóa profile thuê bao. Ở trạng thái hủy, OCS không xử lý giao dịch nào khác.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

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

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18).

---

### 8. Điều chỉnh giao dịch

Gồm 2 lệnh: rollback giao dịch theo requestId và điều chỉnh trực tiếp tài khoản.

#### 8.1. Rollback giao dịch

**Mô tả nghiệp vụ:**

- Rollback giao dịch đã thành công trong thời gian không quá 30 ngày, theo `originalRequestId`, áp dụng cho: nạp tiền (mục 2), trừ lưu lượng (mục 3), cộng lưu lượng Add-On (mục 6) và cộng lưu lượng Pool (mục 10.2).
- Provisioning đảm bảo requestId giữa các lệnh là duy nhất.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

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
| requestDate | Thời điểm thực hiện lệnh (không quá 5 phút so với thời điểm OCS nhận bản tin) | yyyy/MM/dd HH:mm:ss | Có |
| requestId | Mã giao dịch rollback | String | Có |
| originalRequestId | Mã giao dịch gốc cần rollback | String | Có |
| channel | Kênh gửi lệnh, cố định = STARLINK | String | Có |
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| partyCode | Log giao dịch. OCS xuất ra CDR | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_SAME_REQUEST_ID | requestId trùng originalRequestId |
| WS_NOT_FOUND_TRANSACTION | Không tồn tại giao dịch gốc |
| WS_ORIGINAL_TRANS_NOT_SUCCESS | Giao dịch gốc không thành công |
| WS_TRANS_IS_ROLLBACKED | Giao dịch gốc đã được rollback |
| WS_NOT_PERMIT_ROLLBACK | Tài khoản không đủ số dư để rollback giao dịch cộng tiền / cộng lưu lượng |

#### 8.2. Điều chỉnh tài khoản (ModBalance)

**Mô tả nghiệp vụ:**

- Điều chỉnh trực tiếp giá trị, hạn sử dụng của tài khoản tiền (1), tài khoản lưu lượng (11000) hoặc tài khoản Pool (11001) để xử lý khiếu nại, ứng cứu thông tin.

**Request:** `POST http://ipaddress:port/prepayBalanceManagement/v4/balanceAdjust/`

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
| requestDate | Thời điểm thực hiện lệnh (không quá 5 phút so với thời điểm OCS nhận bản tin) | yyyy/MM/dd HH:mm:ss | Có |
| requestId | Mã giao dịch | String | Có |
| channel | Kênh gửi lệnh, cố định = STARLINK | String | Có |
| partyAccount.id | Định danh thuê bao | Xâu ký tự < 30 | Có |
| bucket.id | Mã tài khoản cần điều chỉnh: 1 - tiền; 11000 - lưu lượng; 11001 - Pool | Long | Có |
| balId | Id tài khoản trên OCS (phân biệt các tài khoản cùng balType) | Long | Không |
| amount | Lượng điều chỉnh (+ cộng, - trừ) | Long | Không |
| balance | Set số dư tài khoản | Long | Không |
| addDays | Số ngày gia hạn thêm cho tài khoản | Long | Không |
| expDate | Set hạn sử dụng tài khoản | yyyy/MM/dd HH:mm:ss | Không |
| adjustType | Cố định = ModBalance | String | Có |
| reason | Log giao dịch. OCS xuất ra CDR | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| status | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| BALANCE_NOT_ENOUGH | Số dư không đủ để trừ |

---

### 9. Job/Event nghiệp vụ hết lưu lượng hoặc hết CĐT

**Mô tả nghiệp vụ:** Đây là xử lý chủ động phía OCS (không phải lệnh Provisioning gọi vào):

- **Job gia hạn gói chính:** khi hết mỗi chu kỳ (30 ngày với mã giá 11000000, 1 tháng với mã giá 11000001), OCS tự động gia hạn theo tham số hiện tại của gói chính:
  - Tài khoản tiền ≥ tham số 11002: trừ tài khoản tiền = 11002, reset tài khoản 11000 = tham số 11003, sang chu kỳ mới. Sinh event `RENEW_SUCCESS`.
  - Tài khoản tiền < tham số 11002 (hết CĐT): không gia hạn, tài khoản 11000 về 0, gói chính chuyển trạng thái chờ gia hạn. Sinh event `OUT_OF_CDT` để Provisioning bóp băng thông. Khách hàng nạp tiền (mục 2), sự kiện nạp tiền trên OCS tự động gia hạn gói chính để tiếp tục sử dụng.
- **Event hết lưu lượng:** khi lưu lượng còn lại về 0 trong chu kỳ (mục 3), OCS sinh event `OUT_OF_DATA` để Provisioning bóp băng thông. Khi được cộng lưu lượng Add-On (mục 6) hoặc gia hạn thành công, OCS sinh event `DATA_AVAILABLE` / `RENEW_SUCCESS` để Provisioning mở lại băng thông.

**Bản tin OCS gửi Provisioning (đề xuất):** `POST {URL callback do Provisioning cung cấp}`

```json
{
  "eventId": "OCS_EVT_000001",
  "eventType": "OUT_OF_DATA",
  "eventTime": "2026/11/25 13:45:00",
  "msisdn": "SL0000123456",
  "productExternalId": "11000000",
  "packageType": "LOCAL",
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
| eventType | Loại event: OUT_OF_DATA (hết lưu lượng), OUT_OF_CDT (hết CĐT, không gia hạn được), RENEW_SUCCESS (gia hạn thành công), DATA_AVAILABLE (có lưu lượng trở lại sau khi cộng Add-On) | String | Có |
| eventTime | Thời điểm phát sinh event | yyyy/MM/dd HH:mm:ss | Có |
| msisdn | Định danh thuê bao | Xâu ký tự < 30 | Có |
| productExternalId | Mã giá gói chính hiện tại | Long | Có |
| packageType | Loại gói (giá trị tham số 11001): LOCAL / GLOBAL | String | Có |
| cycleExpDate | Hạn chu kỳ hiện tại của gói chính | yyyy/MM/dd HH:mm:ss | Có |
| moneyBalance | Số dư tài khoản tiền | Long | Có |
| dataBalance | Lưu lượng còn lại (tài khoản 11000) | Long | Có |
| errorCode (response) | 0: Provisioning nhận thành công; khác 0: OCS gửi lại theo cấu hình retry | Int | Có |

---

### 10. Đăng ký gói Pool Starlink

**Mô tả nghiệp vụ:**

- Gói Pool **không quản lý trên OCS**. Sau khi kinh doanh thu tiền mặt của khách hàng doanh nghiệp, Provisioning gọi lên OCS để cộng lưu lượng Pool vào **tài khoản Pool (BalTypeId = 11001)** của thuê bao chủ nhóm.
- Tương tự nhóm doanh nghiệp của Mysign, tài khoản Pool được giữ trên 1 **thuê bao chủ nhóm (thuê bao ảo)**, định danh đề xuất theo định dạng `SL_{mã nhóm/mã hợp đồng}`. Quy trình:
  1. Đấu nối thuê bao chủ nhóm (mục 10.1) — chỉ thực hiện 1 lần.
  2. Cộng lưu lượng Pool vào tài khoản 11001 của thuê bao chủ nhóm (mục 10.2) — mỗi lần khách hàng mua gói Pool, lưu lượng được cộng dồn.
  3. Tạo nhóm và thêm thành viên để chia sẻ lưu lượng Pool (mục 11, 15).

#### 10.1. Đấu nối thuê bao chủ nhóm Pool

**Mô tả nghiệp vụ:** Dùng lệnh đấu nối (mục 1) với mã giá gói chính 11000000, tham số 11002 = 0 và 11003 = 0 (đề xuất), để thuê bao chủ nhóm không bị trừ tiền và không được cộng lưu lượng theo chu kỳ.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "NewConnection",
  "characteristic": [
    { "name": "msisdn", "value": "SL_POOL_000001" },
    { "name": "productExternalId", "value": "11000000" },
    { "name": "imsi", "value": "SL_POOL_000001" },
    { "name": "iccid", "value": "SL_POOL_000001" },
    { "name": "initAmount", "value": "0" },
    { "name": "subType", "value": "13" },
    {
      "name": "characteristicList",
      "value": [
        { "charSpecId": "11001", "stringValue": "LOCAL" },
        { "charSpecId": "11002", "longValue": "0" },
        { "charSpecId": "11003", "longValue": "0" }
      ]
    },
    { "name": "partyCode", "value": "Dau noi chu nhom Pool" }
  ]
}
```

**Response:**

```json
{
  "processFlowSpecification": "NewConnection",
  "processFlowDate": "2026/11/05 10:00:00",
  "characteristic": [
    { "name": "errorCode", "value": "0" },
    { "name": "description", "value": "Success" }
  ]
}
```

**Mô tả Input:**

| Tên trường | Ý nghĩa | Định dạng | Bắt buộc |
|---|---|---|---|
| msisdn | Định danh thuê bao chủ nhóm Pool, định dạng `SL_{mã nhóm}` | Xâu ký tự < 30 | Có |
| productExternalId | Mã giá gói chính (11000000) | Long | Có |
| imsi | Tham số bắt buộc trên OCS, truyền = msisdn | Xâu ký tự | Có |
| iccid | Tham số bắt buộc trên OCS, truyền = msisdn | Xâu ký tự | Có |
| initAmount | Truyền = 0 | Long | Có |
| subType | Loại thuê bao chủ nhóm Pool. Giá trị do VTNET quy hoạch (ví dụ = 13) | Int | Có |
| characteristicList | Tham số gói chính: 11001 = LOCAL/GLOBAL, 11002 = 0, 11003 = 0 | List | Có |
| partyCode | Log giao dịch. OCS xuất ra CDR | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** như mục 1.

#### 10.2. Cộng lưu lượng Pool

**Mô tả nghiệp vụ:**

- Cộng dồn lưu lượng của gói Pool khách hàng đã mua vào tài khoản Pool (11001) của thuê bao chủ nhóm, kèm hạn sử dụng của lưu lượng Pool.
- OCS lưu giao dịch theo `requestId` để tra cứu và rollback (mục 8.1).

**Request:** `POST http://ipaddress:port/prepayBalanceManagement/v4/balanceAdjust/`

```json
{
  "requestDate": "2026/11/05 10:05:00",
  "requestId": "SL_POOL_TOPUP_0001",
  "channel": "STARLINK",
  "partyAccount": { "id": "SL_POOL_000001" },
  "bucket": { "id": "11001" },
  "amount": 10240000,
  "addDays": "",
  "expDate": "2027/11/05 10:00:00",
  "adjustType": "ModBalance",
  "reason": "Cong luu luong Pool 10TB"
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
| requestDate | Thời điểm thực hiện lệnh (không quá 5 phút so với thời điểm OCS nhận bản tin) | yyyy/MM/dd HH:mm:ss | Có |
| requestId | Mã giao dịch, không trùng | String | Có |
| channel | Kênh gửi lệnh, cố định = STARLINK | String | Có |
| partyAccount.id | Định danh thuê bao chủ nhóm Pool | Xâu ký tự < 30 | Có |
| bucket.id | Mã tài khoản Pool, cố định = 11001 | Long | Có |
| amount | Lưu lượng Pool cần cộng (theo đơn vị lưu lượng thống nhất) | Long > 0 | Có |
| addDays | Số ngày sử dụng của lưu lượng Pool tính từ thời điểm hiện tại | Long | Không |
| expDate | Hạn sử dụng của tài khoản Pool. Ưu tiên hơn addDays | yyyy/MM/dd HH:mm:ss | Không |
| adjustType | Cố định = ModBalance | String | Có |
| reason | Log giao dịch (mã gói Pool, mô tả). OCS xuất ra CDR | String | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| status | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_BALTYPE_NOT_ALLOWED | bucket.id không phải tài khoản Pool |

---

### 11. Tạo nhóm thuê bao Starlink

**Mô tả nghiệp vụ:**

- Thuê bao chủ nhóm đã có tài khoản Pool (11001) tạo nhóm chia sẻ lưu lượng Pool (block tổng lưu lượng).
- Cho phép thêm luôn thành viên khi tạo nhóm, mỗi thành viên được cấp hạn mức lưu lượng (`limit`). OCS kiểm tra tổng hạn mức các thành viên không vượt quá lưu lượng Pool.
- Hạn sử dụng tài khoản chia sẻ của thành viên = hạn của tài khoản Pool.
- Id nhóm do OCS sinh và trả về trong response, Provisioning lưu lại để dùng cho các lệnh nhóm.
- Tập lệnh nhóm không kiểm tra trạng thái chặn của thuê bao thành viên; Provisioning kiểm tra trước khi gửi lệnh (giống Mysign).

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

```json
{
  "processFlowSpecification": "CreateGroup",
  "characteristic": [
    { "name": "msisdn", "value": "SL_POOL_000001" },
    { "name": "groupType", "value": "5" },
    { "name": "balType", "value": "11001" },
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
  "processFlowDate": "2026/11/05 10:10:00",
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
| msisdn | Thuê bao chủ nhóm Pool | Xâu ký tự < 30 | Có |
| groupType | Loại nhóm, cố định = 5 (block tổng lưu lượng) | Long | Có |
| balType | Tài khoản dùng để chia sẻ, cố định = 11001 (thay cho mã gói nhóm của Mysign) | Long | Có |
| limit | Tổng lưu lượng chia sẻ của nhóm. Không truyền thì mặc định = số dư tài khoản Pool | Long | Không |
| maxMember | Số thành viên tối đa. Không truyền thì mặc định theo cấu hình OCS | Long | Không |
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

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_MSISDN_NOT_FOUND:+msisdn | Thuê bao thành viên không tồn tại |
| WS_BALANCE_NOT_FOUND | Thuê bao chủ nhóm chưa có tài khoản Pool (11001) |
| WS_LIMIT_MUST_BE_LESS_OR_EQUAL_GROUP_LIMIT | Tổng hạn mức thành viên vượt lưu lượng Pool |
| OVER_MAX_MEMBER | Số thành viên vượt maxMember |

---

### 12. Tra cứu nhóm và thành viên Starlink

**Mô tả nghiệp vụ:**

- Lệnh mới (Mysign chưa có). Tra cứu thông tin nhóm Pool: tổng lưu lượng, lưu lượng chưa cấp, hạn tài khoản Pool, danh sách thành viên và hạn mức/lưu lượng còn lại của từng thành viên.
- Truyền `groupId` để tra cứu 1 nhóm; không truyền thì trả về tất cả nhóm của thuê bao chủ nhóm.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

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
          "limit": "10240000",
          "remainingLimit": "9216000",
          "maxMember": "50",
          "memberCount": "1",
          "effDate": "2026/11/05 10:00:00",
          "expDate": "2027/11/05 10:00:00",
          "listMember": [
            {
              "msisdn": "SL0000123456",
              "limit": "1024000",
              "consumed": "204800",
              "remaining": "819200",
              "expDate": "2027/11/05 10:00:00"
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
| masterMsisdn | Thuê bao chủ nhóm Pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| groupList.groupId | Id nhóm | Long |
| groupList.groupType | Loại nhóm | Int |
| groupList.limit | Tổng lưu lượng chia sẻ của nhóm | Long |
| groupList.remainingLimit | Lưu lượng của nhóm chưa cấp cho thành viên | Long |
| groupList.maxMember | Số thành viên tối đa | Long |
| groupList.memberCount | Số thành viên hiện tại | Long |
| groupList.effDate / expDate | Ngày hiệu lực / hết hạn của tài khoản Pool | yyyy/MM/dd HH:mm:ss |
| listMember.msisdn | Thuê bao thành viên | Xâu ký tự |
| listMember.limit | Hạn mức lưu lượng được cấp | Long |
| listMember.consumed | Lưu lượng đã sử dụng | Long |
| listMember.remaining | Lưu lượng còn lại | Long |
| listMember.expDate | Hạn sử dụng tài khoản chia sẻ của thành viên | yyyy/MM/dd HH:mm:ss |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_GROUP_NOT_EXIST | Không tồn tại nhóm |

---

### 13. Thêm thành viên vào nhóm Starlink

**Mô tả nghiệp vụ:**

- Thêm 1 hoặc nhiều thuê bao Starlink vào nhóm Pool đang hoạt động, mỗi thành viên được cấp hạn mức lưu lượng.
- OCS trừ hạn mức của thành viên vào lưu lượng chưa cấp của nhóm, báo lỗi nếu vượt quá hoặc vượt số thành viên tối đa.
- Hạn sử dụng tài khoản chia sẻ của thành viên = hạn của tài khoản Pool.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

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
| msisdn | Thuê bao chủ nhóm Pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Có |
| listMember | Danh sách thành viên cần thêm | List | Có |
| listMember.msisdn | Thuê bao thành viên | Xâu ký tự < 30 | Có |
| listMember.limit | Hạn mức lưu lượng cấp cho thành viên | Long | Không |
| listMember.shareService | Cố định = 111111111 | String | Có |
| listMember.shareTime.shareTimeType | Cố định = 0 | Int | Có |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_MSISDN_NOT_FOUND:+msisdn | Thuê bao thành viên không tồn tại |
| WS_SHARE_GROUP_NOT_FOUND | Không tồn tại nhóm |
| WS_LIMIT_MUST_BE_LESS_OR_EQUAL_GROUP_LIMIT | Hạn mức vượt lưu lượng chưa cấp của nhóm |
| OVER_MAX_MEMBER | Số thành viên vượt maxMember |
| WS_NOT_PERMISSION | Thuê bao đã là thành viên nhóm |

---

### 14. Hủy thành viên khỏi nhóm Starlink

**Mô tả nghiệp vụ:**

- Loại 1 hoặc nhiều thành viên khỏi nhóm Pool.
- OCS cộng lưu lượng còn lại của thành viên về lưu lượng chưa cấp của nhóm (nếu tài khoản chia sẻ của thành viên còn hạn), sau đó xóa tài khoản chia sẻ của thành viên.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

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
| masterMsisdn | Thuê bao chủ nhóm Pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Có |
| listMemberMsisdn | Danh sách thuê bao thành viên cần hủy | List xâu ký tự | Có |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_MSISDN_NOT_FOUND:+msisdn | Thuê bao thành viên không tồn tại |
| WS_GROUP_NOT_EXIST | Không tồn tại nhóm |
| WS_NOT_FOUND_UPPER_SHARE+memberMsisdn | Thuê bao chưa là thành viên nhóm |

---

### 15. Cập nhật hạn mức thành viên nhóm Starlink

**Mô tả nghiệp vụ:**

- Thay đổi hạn mức lưu lượng cấp cho 1 hoặc nhiều thành viên.
- Provisioning truyền hạn mức mới sau khi thay đổi (ví dụ đang 100 GB, muốn đổi thành 150 GB thì truyền 150 GB theo đơn vị thống nhất).
- Tăng: OCS kiểm tra lưu lượng chưa cấp của nhóm có đủ không, nếu không trả lỗi. Giảm: OCS trừ hạn mức và lưu lượng còn lại của thành viên, cộng phần chênh lệch về lưu lượng chưa cấp của nhóm.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

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
| masterMsisdn | Thuê bao chủ nhóm Pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Có |
| listMember | Danh sách thành viên cần điều chỉnh | List | Có |
| listMember.msisdn | Thuê bao thành viên | Xâu ký tự < 30 | Có |
| listMember.limit | Hạn mức lưu lượng sau khi điều chỉnh | Long ≥ 0 | Có |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_GROUP_NOT_EXIST | Không tồn tại nhóm |
| WS_NOT_FOUND_UPPER_SHARE+memberMsisdn | Thuê bao chưa là thành viên nhóm |
| WS_LIMIT_MUST_BE_LESS_OR_EQUAL_GROUP_LIMIT | Hạn mức tăng vượt lưu lượng chưa cấp của nhóm |

---

### 16. Tra cứu báo cáo sử dụng Pool Starlink

**Mô tả nghiệp vụ:**

- Lệnh mới (Mysign chưa có). Tra cứu báo cáo sử dụng lưu lượng của nhóm Pool tại thời điểm hiện tại: tổng lưu lượng, đã cấp, đã sử dụng, còn lại của nhóm và chi tiết theo từng thành viên.
- Có thể lọc theo 1 thuê bao thành viên.
- Đề xuất: OCS chỉ trả số liệu hiện tại (realtime); báo cáo theo khoảng thời gian trong quá khứ được tổng hợp từ CDR phía CNTT.

**Request:** `POST http://ipaddress:port/processFlowManagement/v4/processFlow/`

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
        "effDate": "2026/11/05 10:00:00",
        "expDate": "2027/11/05 10:00:00",
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
| masterMsisdn | Thuê bao chủ nhóm Pool | Xâu ký tự < 30 | Có |
| groupId | Id nhóm | Long | Có |
| memberMsisdn | Lọc theo 1 thuê bao thành viên. Không truyền thì trả tất cả thành viên | Xâu ký tự < 30 | Không |

**Mô tả Output:**

| Tên trường | Ý nghĩa | Định dạng |
|---|---|---|
| errorCode | Mã lỗi | Int |
| description | Mô tả lỗi | String |
| groupUsage.groupId | Id nhóm | Long |
| groupUsage.effDate / expDate | Ngày hiệu lực / hết hạn của tài khoản Pool | yyyy/MM/dd HH:mm:ss |
| groupUsage.totalLimit | Tổng lưu lượng chia sẻ của nhóm | Long |
| groupUsage.allocated | Tổng hạn mức đã cấp cho thành viên | Long |
| groupUsage.consumed | Tổng lưu lượng thành viên đã sử dụng | Long |
| groupUsage.remaining | Lưu lượng còn lại của nhóm (= totalLimit - consumed) | Long |
| memberUsageList.msisdn | Thuê bao thành viên | Xâu ký tự |
| memberUsageList.limit | Hạn mức được cấp | Long |
| memberUsageList.consumed | Lưu lượng đã sử dụng | Long |
| memberUsageList.remaining | Lưu lượng còn lại | Long |

**Mã lỗi:** Mã lỗi chung (mục 18) và:

| Description | Mô tả |
|---|---|
| WS_GROUP_NOT_EXIST | Không tồn tại nhóm |
| WS_NOT_FOUND_UPPER_SHARE+memberMsisdn | Thuê bao chưa là thành viên nhóm |

---

### 17. CDR cảnh báo dung lượng, thời hạn gói và trạng thái chặn thuê bao Starlink

**Mô tả nghiệp vụ:**

- OCS sinh CDR cảnh báo khi:
  - Lưu lượng còn lại (tài khoản 11000, tài khoản Pool 11001) đạt các ngưỡng cấu hình (ví dụ còn 20%, 0%).
  - Gói chính sắp hết chu kỳ (trước N ngày, cấu hình) và khi hết chu kỳ (gia hạn thành công / hết CĐT).
  - Thuê bao thay đổi trạng thái (chặn/mở theo yêu cầu, hủy).
- CNTT/Provisioning dùng CDR để gửi thông báo khách hàng và đối soát. Ngưỡng cảnh báo và N ngày do kinh doanh quy định, OCS khai báo cấu hình.

**Cấu trúc CDR (đề xuất):**

| STT | Tên trường | Ý nghĩa | Định dạng |
|---|---|---|---|
| 1 | MSISDN | Định danh thuê bao | Xâu ký tự |
| 2 | ALERT_TYPE | Loại cảnh báo: 1 - ngưỡng lưu lượng; 2 - thời hạn gói; 3 - trạng thái thuê bao | Int |
| 3 | ALERT_CODE | Mã chi tiết: DATA_THRESHOLD, OUT_OF_DATA, CYCLE_EXPIRING, RENEW_SUCCESS, OUT_OF_CDT, BLOCK, UNBLOCK, TERMINATE | String |
| 4 | ALERT_TIME | Thời điểm phát sinh | yyyyMMddHHmmss |
| 5 | PRICE_PLAN_ID | Mã giá gói chính (11000000 / 11000001) | Long |
| 6 | PACKAGE_TYPE | Loại gói (tham số 11001): LOCAL / GLOBAL | String |
| 7 | THRESHOLD | Ngưỡng cảnh báo (%) – với ALERT_TYPE = 1 | Int |
| 8 | BAL_TYPE_ID | Mã tài khoản liên quan (1 / 11000 / 11001) | Long |
| 9 | DATA_BALANCE | Lưu lượng còn lại | Long |
| 10 | MONEY_BALANCE | Số dư tài khoản tiền | Long |
| 11 | CYCLE_EXP_DATE | Hạn chu kỳ hiện tại của gói chính | yyyyMMddHHmmss |
| 12 | BEFORE_STATE_SET | Trạng thái trước thay đổi – với ALERT_TYPE = 3 | Xâu ký tự |
| 13 | NOW_STATE_SET | Trạng thái sau thay đổi – với ALERT_TYPE = 3 | Xâu ký tự |
| 14 | GROUP_ID | Id nhóm Pool (nếu là thành viên/chủ nhóm) | Long |

---

### 18. Mã lỗi chung

Áp dụng cho tất cả các lệnh ở trên (mã lỗi đặc thù được nêu trong từng lệnh):

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
