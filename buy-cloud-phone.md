# Tài liệu API BuyCloudPhone

> **Base URL:** `https://meows.vn`  
> **Phiên bản:** 1.0.0  
> **Cập nhật lần cuối:** 23-03-2026

## Tổng quan

Tài liệu này mô tả các endpoint API cho tính năng **BuyCloudPhone** trên [meows.vn](https://meows.vn). Dịch vụ BuyCloudPhone cho phép người dùng mua điện thoại cloud từ nhiều nhà cung cấp bao gồm Vsphone, Ugphone, Vmos và LD Cloud.

---

## Mục lục

1. [Xác thực](#xác-thực)
2. [API Trạng thái Hệ thống](#api-trạng-thái-hệ-thống)
3. [API Mua Cloud Phone](#api-mua-cloud-phone)
4. [API Trạng thái Task](#api-trạng-thái-task)
5. [API Lịch sử](#api-lịch-sử)
6. [Mô hình Dữ liệu](#mô-hình-dữ-liệu)
7. [Mã lỗi](#mã-lỗi)
8. [Định dạng Tài khoản](#định-dạng-tài-khoản)

---

## Xác thực

Hầu hết các endpoint yêu cầu xác thực qua cookie. Hệ thống sử dụng middleware sau:

- **`checkUserCookie`**: Xác thực phiên người dùng qua cookie `.COOKIESECURITY`

### Yêu cầu Cookie

| Tên Cookie | Mục đích | Bắt buộc |
|------------|----------|----------|
| `.COOKIESECURITY` | Token xác thực người dùng | Có (cho các endpoint được bảo vệ) |

---

## API Trạng thái Hệ thống

### GET /api/status

Lấy trạng thái hiện tại của các dịch vụ hệ thống.

#### Request

```http
GET /api/status
Host: meows.vn
```

#### Response

```json
{
  "auto_buyphone": {
    "status": "Online" | "Offline"
  },
  "website": {
    "status": "Online" | "Offline"
  }
}
```

#### Ví dụ

```bash
curl -X GET https://meows.vn/api/status
```

#### Response Ví dụ

```json
{
  "auto_buyphone": {
    "status": "Online"
  },
  "website": {
    "status": "Online"
  }
}
```

---

## API Mua Cloud Phone

### POST /api/buy-cloud-phone

Gửi một task mua cloud phone vào hàng đợi. Đây là endpoint **được bảo vệ** yêu cầu xác thực.

#### Request

```http
POST /api/buy-cloud-phone
Host: meows.vn
Content-Type: application/json
Cookie: .COOKIESECURITY=<user_token>
```

#### Request Body

| Trường | Kiểu | Bắt buộc | Mô tả |
|--------|------|----------|-------|
| `service` | string | Có | Tên dịch vụ cloud phone (`Vsphone`, `Ugphone`, `Vmos`, `LD Cloud`) |
| `accounts` | array | Có | Mảng các object tài khoản (xem Định dạng Tài khoản bên dưới) |
| `region` | string | Không | Khu vực cho dịch vụ Ugphone (`Hong Kong`, `Singapore`, `Japan`, `Germany`, `America`) |

#### Định dạng Tài khoản theo Dịch vụ

**Cho Vsphone/Vmos:**
```json
{
  "accounts": [
    {
      "account": "user@example.com",
      "password": "password123"
    }
  ]
}
```

**Cho Ugphone/LD Cloud:**
```json
{
  "accounts": [
    {
      "cookie": "raw_cookie_string_or_fetch_atob_script"
    }
  ],
  "region": "Singapore"
}
```

#### Response

```json
{
  "success": true,
  "taskId": "string",
  "historyId": "string",
  "queuePosition": number,
  "message": "Task đã được thêm vào queue"
}
```

#### Response Lỗi

```json
{
  "success": false,
  "error": "Thiếu trường service."
}
```

```json
{
  "success": false,
  "error": "Service không hỗ trợ: InvalidService"
}
```

```json
{
  "success": false,
  "error": "Thiếu tài khoản cho Vsphone."
}
```

#### Ví dụ

```bash
curl -X POST https://meows.vn/api/buy-cloud-phone \
  -H "Content-Type: application/json" \
  -b ".COOKIESECURITY=your_token" \
  -d '{
    "service": "Vsphone",
    "accounts": [
      {
        "account": "user@example.com",
        "password": "password123"
      }
    ]
  }'
```

---

## API Trạng thái Task

### GET /api/buy-cloud-phone/status/:taskId

Kiểm tra trạng thái của một task mua. Endpoint này **không** yêu cầu xác thực.

#### Request

```http
GET /api/buy-cloud-phone/status/:taskId
Host: meows.vn
```

#### URL Parameters

| Parameter | Kiểu | Mô tả |
|-----------|------|-------|
| `taskId` | string | Task ID được trả về từ API submit |

#### Response

```json
{
  "success": true,
  "status": "pending" | "processing" | "completed" | "failed",
  "result": {
    // Dữ liệu kết quả mua
  },
  "error": "error message if failed",
  "historyId": "string",
  "service": "string",
  "createdAt": "ISO8601 timestamp",
  "completedAt": "ISO8601 timestamp or null"
}
```

#### Response Lỗi

```json
{
  "success": false,
  "error": "Task không tồn tại",
  "taskId": "string"
}
```

#### Ví dụ

```bash
curl -X GET https://meows.vn/api/buy-cloud-phone/status/abc123xyz
```

#### Response Ví dụ

```json
{
  "success": true,
  "status": "completed",
  "result": {
    "success": true,
    "orderId": "ORDER123",
    "message": "Purchase successful"
  },
  "historyId": "hist_abc123",
  "service": "Vsphone",
  "createdAt": "2026-03-23T12:00:00.000Z",
  "completedAt": "2026-03-23T12:00:05.000Z"
}
```

---

## API Lịch sử

### GET /api/buy-cloud-phone/history

Lấy lịch sử mua hàng cho người dùng đã xác thực. Đây là endpoint **được bảo vệ** yêu cầu xác thực.

#### Request

```http
GET /api/buy-cloud-phone/history?limit=50
Host: meows.vn
Cookie: .COOKIESECURITY=<user_token>
```

#### Query Parameters

| Parameter | Kiểu | Mặc định | Mô tả |
|-----------|------|----------|-------|
| `limit` | number | 50 | Số lượng tối đa các mục lịch sử cần trả về |

#### Response

```json
{
  "success": true,
  "history": [
    {
      "id": "string",
      "cloudName": "Vsphone" | "Ugphone" | "Vmos" | "LD Cloud",
      "account": "string",
      "password": "string",
      "region": "string" | null,
      "status": "pending" | "processing" | "success" | "failed",
      "amountId": "string" | null,
      "orderId": "string" | null,
      "message": "string",
      "timestamp": "ISO8601 timestamp",
      "createdAt": "ISO8601 timestamp",
      "updatedAt": "ISO8601 timestamp"
    }
  ]
}
```

#### Ví dụ

```bash
curl -X GET "https://meows.vn/api/buy-cloud-phone/history?limit=10" \
  -b ".COOKIESECURITY=your_token"
```

---

### DELETE /api/buy-cloud-phone/history/:id

Xóa một mục lịch sử cụ thể. Đây là endpoint **được bảo vệ** yêu cầu xác thực.

#### Request

```http
DELETE /api/buy-cloud-phone/history/:id
Host: meows.vn
Cookie: .COOKIESECURITY=<user_token>
```

#### URL Parameters

| Parameter | Kiểu | Mô tả |
|-----------|------|-------|
| `id` | string | ID của mục lịch sử cần xóa |

#### Response

```json
{
  "success": true,
  "history": [
    // Các mục lịch sử còn lại
  ]
}
```

#### Response Lỗi

```json
{
  "success": false,
  "error": "Unauthorized",
  "message": "Bạn cần đăng nhập để xóa lịch sử"
}
```

#### Ví dụ

```bash
curl -X DELETE https://meows.vn/api/buy-cloud-phone/history/hist_abc123 \
  -b ".COOKIESECURITY=your_token"
```

---

### DELETE /api/buy-cloud-phone/history

Xóa toàn bộ lịch sử mua hàng cho người dùng đã xác thực. Đây là endpoint **được bảo vệ** yêu cầu xác thực.

#### Request

```http
DELETE /api/buy-cloud-phone/history
Host: meows.vn
Cookie: .COOKIESECURITY=<user_token>
```

#### Response

```json
{
  "success": true,
  "message": "History đã được xóa"
}
```

#### Ví dụ

```bash
curl -X DELETE https://meows.vn/api/buy-cloud-phone/history \
  -b ".COOKIESECURITY=your_token"
```

---

## Mô hình Dữ liệu

### Trạng thái Task

| Trạng thái | Mô tả |
|------------|-------|
| `pending` | Task đang chờ trong hàng đợi |
| `processing` | Task đang được xử lý |
| `completed` | Task hoàn thành thành công |
| `failed` | Task thất bại |

### Trạng thái Lịch sử

| Trạng thái | Mô tả |
|------------|-------|
| `pending` | Đang chờ xử lý |
| `processing` | Đang được xử lý |
| `success` | Mua hàng thành công |
| `failed` | Mua hàng thất bại |

### Loại Dịch vụ

| Dịch vụ | Mô tả | Loại Tài khoản |
|---------|-------|----------------|
| `Vsphone` | Dịch vụ cloud phone cao cấp | Email + Mật khẩu |
| `Ugphone` | Cloud phone đa khu vực | Cookie |
| `Vmos` | Nền tảng hệ điều hành di động ảo | Email + Mật khẩu |
| `LD Cloud` | Dịch vụ LD Cloud phone | Cookie |

### Mã Khu vực (cho Ugphone)

| Khu vực | Mã |
|---------|-----|
| Hồng Kông | `Hong Kong` |
| Singapore | `Singapore` |
| Nhật Bản | `Japan` |
| Đức | `Germany` |
| Mỹ | `America` |

---

## Mã lỗi

| Mã lỗi | Thông báo | Mô tả |
|--------|-----------|-------|
| 400 | `Thiếu trường service.` | Thiếu trường service |
| 400 | `Service không hỗ trợ: {service}` | Tên service không hợp lệ |
| 400 | `Thiếu tài khoản cho {service}.` | Thiếu dữ liệu tài khoản |
| 400 | `Tài khoản {n} thiếu account hoặc password cho {service}.` | Định dạng tài khoản không hợp lệ |
| 400 | `Tài khoản {n} thiếu cookie cho {service}.` | Thiếu cookie cho dịch vụ dựa trên cookie |
| 400 | `Tài khoản {n} cookie lỗi: {error}` | Định dạng cookie không hợp lệ |
| 401 | `Unauthorized` | Chưa xác thực |
| 401 | `Bạn cần đăng nhập để xem lịch sử` | Yêu cầu đăng nhập |
| 404 | `Task không tồn tại` | Task ID không tìm thấy |
| 500 | `Lỗi server không xác định` | Lỗi server nội bộ |

---

## Định dạng Tài khoản

### Vsphone / Vmos

Chấp nhận các định dạng sau:

```
# Định dạng 1: email+password
user@example.com+password123

# Định dạng 2: email|password
user@example.com|password123

# Định dạng 3: username+password
myuser+password123

# Định dạng 4: username|password
myuser|password123
```

### Ugphone / LD Cloud

Yêu cầu chuỗi cookie thô hoặc script fetch với mã hóa atob:

```
# Cookie thô
cookie_raw_string_here

# Hoặc script fetch atob
fetch(atob('aHR0cHM6Ly9...')).then(res => res.text()).then(eval)
```

---

## Giới hạn tốc độ

- Hiện tại không có giới hạn tốc độ nào được áp dụng cho các endpoint này
- Việc kiểm tra trạng thái task nên được thực hiện với khoảng thời gian phù hợp (khuyến nghị: 1-2 giây)

---

## Hỗ trợ

Nếu có vấn đề hoặc câu hỏi, vui lòng liên hệ với đội phát triển.
