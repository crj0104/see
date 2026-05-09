# 数据库设计文档 (DATABASE_DESIGN)

本项目遵循**“微服务数据库物理隔离”**的最佳实践。系统拆分为 4 个独立的 MySQL 数据库，确保各服务的数据与领域边界互不侵入。

---

## 1. 核心交易库 (`payment_transaction_db`)

所属微服务：`payment-transaction-service`
核心职责：存储支付单、退款单、接收的回调日志，以及用于解决分布式事务的 Outbox 表。

### 1.1 支付交易流水表 (`payment_transaction`)
记录商户发起的每一笔支付请求及流转状态。
*   `id` (PK): 自增主键
*   `transaction_id` (UK): 中台全局唯一的支付流水号（如 `PAY20260509...`）
*   `order_id` (Index): 商户业务订单号
*   `amount`: 支付金额 (Decimal)
*   `currency`: 币种 (默认 CNY)
*   `payment_method`: 支付方式 (WECHAT_PAY, ALIPAY)
*   `status` (Index): 支付状态 (INIT, PROCESSING, SUCCESS, FAILED, CLOSED)
*   `provider_transaction_id`: 第三方渠道（微信/支付宝）返回的真实流水号
*   `notify_url`: 商户异步接收结果的地址
*   `raw_callback_data`: 渠道回调的原始报文备份
*   `created_at`, `updated_at`: 时间戳

### 1.2 退款记录表 (`refund_record`)
记录每一笔退款流水，支持同一笔支付单多次部分退款。
*   `id` (PK)
*   `refund_id` (UK): 中台全局唯一的退款流水号
*   `transaction_id` (Index): 关联的支付流水号
*   `amount`: 退款金额
*   `status`: 退款状态 (INIT, PROCESSING, SUCCESS, FAILED)
*   `provider_refund_id`: 第三方渠道的退款单号

### 1.3 回调与审计日志表 (`callback_log`)
记录与第三方交互的每一条入栈(INBOUND)和出栈(OUTBOUND)日志。
*   `transaction_id` (Index): 关联流水号
*   `callback_direction`: 方向 (INBOUND: 第三方发给中台 / OUTBOUND: 中台发给第三方)
*   `request_body`, `response_body`: 报文存档
*   `callback_status`, `retry_count`: 处理状态与重试次数

### 1.4 本地消息表 (`outbox_entity`)
**核心！** 解决分布式事务（如落库成功但发 MQ 失败）的本地消息表。
*   `aggregate_id`: 关联聚合根 (通常是 transaction_id)
*   `event_type`: 事件类型 (如 `payment.success`)
*   `payload`: 将要发送给 RabbitMQ 的 JSON 消息体
*   `status` (Index): 投递状态 (PENDING, PROCESSED, FAILED)

---

## 2. 异步通知库 (`payment_notification_db`)

所属微服务：`payment-notification-service`
核心职责：解耦商户通知逻辑，专门负责向商户发送 HTTP 回调并处理阶梯重试。

### 2.1 商户通知记录表 (`notification_record`)
*   `transaction_id` (Index): 支付流水号
*   `notify_url`: 目标地址
*   `notify_type`: 通知类型 (PAYMENT_RESULT)
*   `notify_content`: 推送的 JSON
*   `notification_status` (Index): 通知状态 (PENDING, SUCCESS, FAILED)
*   `retry_count`, `max_retry`, `next_retry_time`: **核心调度字段**，用于实现 `1m, 5m, 1h` 的阶梯重试。
*   `response_content`: 商户返回的结果（必须为 SUCCESS 才算成功）

---

## 3. 对账异常库 (`payment_reconciliation_db`)

所属微服务：`payment-reconciliation-service`
核心职责：每日凌晨 T+1 对账，记录对账批次与发现的差错。

### 3.1 对账批次表 (`reconciliation_batch`)
*   `batch_no` (UK): 批次号 (如 `20260509-WECHAT`)
*   `provider`: 渠道 (WECHAT_PAY)
*   `bill_date`: 账单日期
*   `status`: 批次状态 (DOWNLOADING, COMPARING, FINISHED)
*   `total_count`, `anomaly_count`: 笔数统计

### 3.2 对账差错明细表 (`reconciliation_anomaly`)
*   `batch_no` (Index): 关联批次
*   `transaction_id` (Index): 本地流水号
*   `anomaly_type`: 异常类型 (长款-平台无渠道有 / 短款-平台有渠道无 / 金额不符)
*   `local_amount`, `provider_amount`: 金额快照
*   `resolution_status`: 处理状态 (UNRESOLVED, RESOLVED) 供财务人员在后台操作平账。

---

## 4. 运营后台库 (`payment_admin_db`)

所属微服务：`payment-admin-service`
核心职责：存储商户的接入配置、网关鉴权密钥、第三方渠道参数。

### 4.1 商户基础信息表 (`merchant_info`)
*   `merchant_no` (UK): 内部商户号
*   `merchant_name`: 商户名称
*   `api_key` (UK): 分发给商户的 API-Key（网关鉴权）
*   `api_secret`: 分发给商户的 Secret（HMAC-SHA256 签名计算）
*   `status`: 商户状态 (ACTIVE, FROZEN)

### 4.2 商户渠道配置表 (`merchant_channel_config`)
*   `merchant_no`: 关联商户
*   `channel_code`: 渠道 (WECHAT_PAY)
*   `provider_mch_id`: 第三方分配的商户号 (如微信的 mch_id)
*   `config_json`: 敏感配置字典 (存放证书路径、APIv3 密钥等 JSON 数据)

---
> 备注：所有 DDL 建表脚本已存放在项目的 `db/mysql/init/` 目录下，支持 Docker 一键初始化。