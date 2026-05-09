# Distributed Payment Core Platform — 架构设计文档

> 核心技术栈：Spring Boot 3.2.3 + Java 21 + Spring Cloud Alibaba + MyBatis-Plus + RabbitMQ

---

## 一、系统架构图

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                             外部客户端 / 商户业务系统                         │
│                              HTTPS api.pay.com                              │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────────────────────────────┐
│                         payment-api-gateway                                 │
│                    Spring Cloud Gateway :8080                                │
│                                                                             │
│  ├─ ApiSignatureAuthFilter: HMAC-SHA256签名验签 / API-Key校验 / 防重放(Nonce) │
│  ├─ RouteConfig:  /api/v1/payments/*         → lb://payment-transaction     │
│  │               /api/v1/callbacks/provider → lb://payment-channel         │
│  │               /api/v1/admin/*            → lb://payment-admin           │
│  └─ RequestLogFilter: X-Trace-Id 链路追踪                                   │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
┌─────────▼──────────┐   ┌─────────▼──────────┐   ┌─────────▼──────────┐
│ payment-           │   │ payment-           │   │ payment-           │
│ transaction-service│   │ channel-service    │   │ admin-service      │
│ :8081              │   │ :8085              │   │ :8089              │
│ 核心交易大脑        │   │ 渠道防腐层 (ACL)    │   │ 运营管理 BFF       │
├────────────────────┤   ├────────────────────┤   ├────────────────────┤
│ 状态机流转          │   │ 策略/工厂模式加载SDK│   │ 商户 API-Key 管理  │
│ Outbox 消息持久化   │   │ 解析原生 XML/JSON   │   │ 订单查询 / 强制退款│
│ 风控责任链拦截      │   │ 原生加密 / 验签     │   │                    │
└─────────┬──────────┘   └─────────┬──────────┘   └─────────┬──────────┘
          │ (MQ)                    │ (MQ)                    │ (Feign)
          ├─────────────────────────┘                         │
┌─────────▼──────────┐   ┌────────────────────┐               │
│ payment-           │   │ payment-           │◄──────────────┘
│ notification-      │   │ reconciliation-    │
│ service :8083      │   │ service :8082      │
│ 异步通知服务        │   │ 自动对账服务        │
├────────────────────┤   ├────────────────────┤
│ 消费 MQ 投递商户    │   │ 5 类异常扫描        │
│ 阶梯指数退避重试    │   │ T+1 拉取第三方账单  │
└────────────────────┘   └────────────────────┘
```

---

## 二、关键企业级设计决策

### 2.1 渠道防腐层 (Anti-Corruption Layer)
**痛点**：传统支付系统将微信、支付宝的 SDK 直接写在交易代码中，导致“Jar包地狱”，且任何一家 API 升级都会导致整个核心系统重启。
**方案**：彻底剥离。引入 `payment-channel-service`，交易服务仅通过 OpenFeign 与其进行标准 DTO 通信。收到外部异步回调时，渠道服务负责解密并发送标准化 MQ 给交易服务。

### 2.2 Outbox Pattern (本地消息表) —— 100% 最终一致性
**痛点**：支付成功后，MySQL 状态更新了，但往 RabbitMQ 发通知消息时网络闪断，导致商户永远收不到回调。
**方案**：
1. `payment-transaction-service` 在同一个数据库事务中，更新 `payment_transaction` 表，并向 `outbox_entity` 表插入一条 PENDING 的消息记录。
2. 事务提交。
3. 后台有一个高频定时任务（或 Canal 监听 binlog），扫描 `outbox_entity`，将消息稳妥投递到 RabbitMQ，收到 ACK 后将状态改为 PROCESSED。即使 MQ 宕机，消息也不会丢失。

### 2.3 责任链模式 (Chain of Responsibility) —— 风控引擎
在订单落库前，挂载了 `RiskControlManager`。
内部包含 `BlacklistRiskRule`（黑名单拦截）、`HighFrequencyRiskRule`（IP高频防刷）、`LargeAmountRiskRule`（大额熔断）。新增风控规则只需实现 `RiskRule` 接口，对核心流程零侵入。

### 2.4 三重防击穿与防覆盖
1. **防瞬间重放**：Gateway 层的 `X-Nonce` + Redis `SETNX`，拦截绝对重复请求。
2. **防业务重放**：Service 层的 `@Idempotent` 切面与 `idempotentKey`，以及 MySQL 的 `UNIQUE KEY`。
3. **防状态覆盖**：MySQL 乐观锁，`UPDATE payment_transaction SET status = 'SUCCESS' WHERE transaction_id = ? AND status = 'PROCESSING'`。

---

## 三、微服务数据库隔离边界

| 微服务 | 数据库名 | 核心表 |
| --- | --- | --- |
| **Transaction** | `payment_transaction_db` | `payment_transaction`, `refund_record`, `outbox_entity` |
| **Notification**| `payment_notification_db`| `notification_record` |
| **Reconciliation**| `payment_reconciliation_db`| `reconciliation_batch`, `reconciliation_anomaly` |
| **Admin**       | `payment_admin_db`       | `merchant_info`, `merchant_channel_config` |

所有数据库均通过 Docker 的 `docker-entrypoint-initdb.d` 自动初始化。
