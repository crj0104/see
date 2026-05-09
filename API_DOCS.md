# 分布式支付中台 API 接口文档

## 1. 全局鉴权规范 (API Signature)

为了保证内部系统调用的绝对安全，所有经过 API 网关的请求（除 `/api/v1/callbacks/provider/*` 等被动接收外部回调的白名单外）均需进行 **HMAC-SHA256 签名鉴权**。

### 1.1 必传 Header 列表
业务方在发起 HTTP 请求时，必须在 Header 中携带以下 4 个参数：
*   `X-Api-Key`: 中台分配的唯一客户端标识（如 `AK_abcd1234`）。
*   `X-Timestamp`: 发起请求时的毫秒级时间戳（与中台服务器时间误差不得超过 5 分钟）。
*   `X-Nonce`: 随机字符串（UUID）。同一 ApiKey 下 5 分钟内不可重复，用于防重放攻击。
*   `X-Signature`: 计算出的请求签名串。

### 1.2 签名计算规则
1. **构造规范化负载 (Canonical Payload)**
   将以下字段使用换行符 `\n` 拼接：
   ```text
   HTTP_METHOD
   HTTP_PATH (如 /api/v1/payments)
   QUERY_STRING (如 orderId=123，若无则留空)
   X-Timestamp
   X-Nonce
   BODY_SHA256_HEX (对请求 Body 的字节流做 SHA-256 并转为 Hex 字符串；若无 Body 则是空字节流的 SHA-256)
   ```
2. **计算 HMAC-SHA256**
   使用中台分配的 `Secret` 对上述 Payload 进行 HMAC-SHA256 运算，输出 Base64 字符串。

---

## 2. 核心交易接口 (面向商户)

### 2.1 创建支付单 (统一下单)
*   **接口**: `POST /api/v1/payments`
*   **请求体**:
    ```json
    {
      "orderId": "BIZ202405090001",
      "amount": 100.50,
      "currency": "CNY",
      "paymentMethod": "WECHAT_PAY", // WECHAT_PAY, ALIPAY
      "idempotentKey": "idem_12345abcde", 
      "description": "购买超级大会员",
      "notifyUrl": "https://biz-system.com/api/payment/notify",
      "clientIp": "192.168.1.100"
    }
    ```
*   **响应 (201 Created)**: 返回 `transactionId`。

### 2.2 执行支付 (拉起收银台)
*   **接口**: `POST /api/v1/payments/{transactionId}/execute`
*   **描述**: 根据创建好的流水号，实际调用底层第三方支付渠道。
*   **响应**: 包含用于前端唤起微信/支付宝的 SDK 参数或二维码链接。

### 2.3 查询支付结果
*   **接口**: `GET /api/v1/payments/{transactionId}` 或 `GET /api/v1/payments?orderId={orderId}`

---

## 3. 运营管理后台接口 (面向内部 Admin)

> 注意：调用 Admin 接口需要使用拥有 Admin 权限的 API-Key。

### 3.1 创建新商户
*   **接口**: `POST /api/v1/admin/merchants?merchantName={name}`
*   **响应**: 自动生成并返回 `apiKey` 和 `apiSecret`。

### 3.2 强制关闭支付单
*   **接口**: `POST /api/v1/admin/transactions/{transactionId}/close`
*   **描述**: 当订单卡死在 PENDING 时，客服人员手工干预强制关单。

### 3.3 强制重试回调
*   **接口**: `POST /api/v1/admin/transactions/{transactionId}/retry`
*   **描述**: 当回调失败且自动重试次数耗尽时，运维人员手工触发。

---

## 4. 回调与通知接口 (Webhook)

### 4.1 接收第三方网关回调 (中台被动接收)
*   **接口**: `POST /api/v1/callbacks/provider/{provider}`
*   **说明**: 供微信/支付宝调用。网关已将其加入鉴权白名单，直接路由至 `payment-channel-service` 进行底层解密验签。

### 4.2 异步通知商户系统 (中台主动发出)
*   **触发时机**: 当中台确认支付成功后。
*   **请求方式**: `POST` (发送至业务方建单时传入的 `notifyUrl`)
*   **请求体格式**:
    ```json
    {
      "transactionId": "PAY6F8A9B2C...",
      "orderId": "BIZ202405090001",
      "status": "SUCCESS",
      "timestamp": "2026-05-09T12:00:00Z"
    }
    ```
*   **商户要求**: 商户系统处理成功必须返回字符串 `"SUCCESS"`。若返回其他内容或超时，中台将执行阶梯退避重试（最大 5 次）。
