# API Documentation: Buy Cloud Phone

## Endpoint
```http
POST https://meows.io.vn/api/buy-cloud-phone
```

## Description
Mua máy chủ ảo (cloud phone) từ các dịch vụ free. API này xử lý mua máy theo hàng đợi.

## Rate Limiting
- Endpoint này được bảo vệ bởi middleware `buyphonelitmit`
- Kiểm tra rate limit trước khi xử lý request

## Request

### Headers
```json
{
  "Content-Type": "application/json"
}
```

### Request Body
```json
{
  "service": "Vsphone|Ugphone",
  "region": "Hong Kong|Singapore|Japan|Germany|America", // Chỉ cần cho Ugphone
  "accounts": [
    // Cho Vsphone
    {
      "token": "string",
      "userId": "string"
    },
    // Hoặc cho Ugphone  
    {
      "cookie": "string"
    }
  ]
}
```

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `service` | string | Dịch vụ mua máy. Giá trị: `"Vsphone"` hoặc `"Ugphone"` |
| `region` | string  | Khu vực máy chủ. **Bắt buộc với Ugphone**, bỏ qua với Vsphone |
| `accounts` | array  | Danh sách tài khoản để thử mua |

#### Account Structure

**Vsphone Account:**
```json
{
  "token": "an6Duq7yrsHVVZFgNcx6GWjAV8vErFsd",
  "userId": "1744880"
}
```

**Ugphone Account:**
```json
{
  "cookie": "session_data_or_json_string"
}
```

### Supported Regions (Ugphone only)
- `Hong Kong` 🇭🇰
- `Singapore` 🇸🇬
- `Japan` 🇯🇵  
- `Germany` 🇩🇪
- `America` 🇺🇸

## Response

### Success Response (200 OK)
```json
{
  "success": true,
  "message": "ĐÃ MUA ĐƯỢC MÁY !!!",
  "amount_id": "12345",
  "service": "Vsphone",
  "region": "Hong Kong", // Chỉ có với Ugphone
  "tried": 3,
  "accountUsed": 2,
  "retryUsed": 1,
  "totalResults": 3,
  "proxy": ["proxy1", "proxy2"],
  "queuePosition": 1
}
```

### Error Response (400 Bad Request)
```json
{
  "success": false,
  "error": "Thiếu trường service."
}
```

### Error Response (500 Internal Server Error)
```json
{
  "success": false,
  "error": "Lỗi server không xác định",
  "message": "Connection timeout"
}
```

## Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | Trạng thái thành công/thất bại |
| `message` | string | Thông báo kết quả |
| `amount_id` | string | ID của máy đã mua (chỉ khi thành công) |
| `service` | string | Dịch vụ đã sử dụng |
| `region` | string | Khu vực máy chủ (chỉ Ugphone) |
| `tried` | number | Tổng số lần thử |
| `accountUsed` | number | Thứ tự tài khoản thành công |
| `retryUsed` | number | Số lần retry với tài khoản thành công |
| `totalResults` | number | Số kết quả trả về từ API gốc |
| `proxy` | array | Danh sách proxy đã sử dụng |
| `queuePosition` | number | Vị trí trong hàng đợi |
| `error` | string | Mô tả lỗi (chỉ khi thất bại) |

## Examples

### Example 1: Mua Vsphone
```bash
curl -X POST http://your-domain.com/api/buy-cloud-phone \
  -H "Content-Type: application/json" \
  -d '{
    "service": "Vsphone",
    "accounts": [
      {
        "token": "an6Duq7yrsHVVZFgNcx6GWjAV8vErFsd",
        "userId": "1744880"
      },
      {
        "token": "another_token_here",
        "userId": "1234567"
      }
    ]
  }'
```

**Response:**
```json
{
  "success": true,
  "message": "ĐÃ MUA ĐƯỢC MÁY !!!",
  "amount_id": "VS12345",
  "service": "Vsphone",
  "accountUsed": 1,
}
```

### Example 2: Mua Ugphone
```bash
curl -X POST http://your-domain.com/api/buy-cloud-phone \
  -H "Content-Type: application/json" \
  -d '{
    "service": "Ugphone", 
    "region": "Singapore",
    "accounts": [
      {
        "cookie": "{\"session\":\"abc123\",\"user\":\"test\"}"
      }
    ]
  }'
```

**Response:**
```json
{
  "success": true,
  "message": "ĐÃ MUA ĐƯỢC MÁY !!!",
  "amount_id": "UG67890",
  "service": "Ugphone",
  "region": "Singapore", 
  "accountUsed": 1,
}
```

### Example 3: Error - Missing Service
```bash
curl -X POST http://your-domain.com/api/buy-cloud-phone \
  -H "Content-Type: application/json" \
  -d '{
    "accounts": [{"token": "test", "userId": "123"}]
  }'
```

**Response (400):**
```json
{
  "success": false,
  "error": "Thiếu trường service."
}
```

## Error Codes

| HTTP Code | Error | Description |
|-----------|-------|-------------|
| 400 | `Thiếu trường service.` | Không có field service trong request |
| 400 | `Service không hỗ trợ: {service}` | Service không phải Vsphone/Ugphone |
| 400 | `Thiếu tài khoản cho {service}.` | Array accounts rỗng hoặc null |
| 400 | `Tài khoản {i} thiếu token hoặc userId cho Vsphone.` | Account Vsphone thiếu thông tin |
| 400 | `Tài khoản {i} thiếu cookie cho Ugphone.` | Account Ugphone thiếu cookie |
| 500 | `Lỗi server không xác định` | Lỗi trong quá trình xử lý |

## Queue System

API sử dụng hệ thống hàng đợi (queue) để xử lý các yêu cầu:

- **Concurrent threads:** 3 threads xử lý đồng thời
- **Queue position:** Trả về vị trí trong hàng đợi
- **FIFO processing:** Xử lý theo thứ tự vào trước ra trước

## Retry Logic

### Per Account Retry
- **Vsphone:** Tối đa 2 lần retry/account
- **Ugphone:** Tối đa 2 lần retry/account  
- **BBcloud:** Tối đa 4 lần retry/account

### Account Fallback
- Thử tuần tự từng account cho đến khi thành công
- Nếu account 1 thất bại → chuyển sang account 2
- Nếu tất cả accounts thất bại → trả về lỗi

### Smart Error Handling
- Lỗi region/hết máy → skip retry, chuyển account tiếp
- Lỗi account/auth → retry với cùng account
- Timeout → retry với delay

## Implementation Notes

1. **Validation Order:**
   - Service validation
   - Accounts validation  
   - Account structure validation

2. **Processing Flow:**
   ```
   Request → Validation → Queue → Thread Pool → Retry Logic → Response
   ```

3. **Thread Safety:** 
   - Queue được shared giữa các threads
   - Thread status tracking tránh race conditions

4. **Memory Management:**
   - Không sử dụng localStorage/sessionStorage
   - State được lưu trong memory during session

## Rate Limiting Recommendations

- Implement proper rate limiting để tránh spam
- Monitor queue length để tránh memory issues
- Set timeout reasonable cho các external API calls

## Security Considerations  

- Validate và sanitize tất cả input
- Không log sensitive data (tokens, cookies)  
- Implement proper error handling để tránh information disclosure
- Consider implementing authentication/authorization
