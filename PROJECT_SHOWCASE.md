# 🚀 Distributed Payment Core Platform (分布式支付中台) 深度解析指南

> **写给其他 AI 或技术评审专家的导读说明**：
> 这是一份面向架构师、AI 编程助手或技术面试官的“上帝视角”文档。本项目并非简单的 CRUD 拼凑，而是严格按照**大型互联网公司支付中台标准**进行重构和演进的系统。
> 本文档将深入剖析系统的模块划分、核心痛点解决思路、设计模式的运用以及关键技术选型。

---

## 🌟 一、系统定位与价值

在传统的单体架构中，业务线（如电商、外卖）往往直接在自己的代码里引入微信支付、支付宝的 SDK，这会导致几个致命问题：
1. **Jar 包冲突（依赖地狱）**：不同渠道的加密包版本不一。
2. **重复造轮子**：每个业务线都要自己写防掉单、写退款、写对账。
3. **资金安全风险**：业务线开发水平参差不齐，极易出现高频刷单、并发导致订单状态被覆盖。

**本项目（支付中台）的价值**：
彻底将“支付”这一领域能力下沉为**基础设施**。上游业务系统（商户）只需通过标准 HTTP + 鉴权对接中台，中台负责解决所有底层复杂性（加解密、网络抖动补偿、幂等、对账、风控）。

---

## 🏗️ 二、微服务矩阵拆解 (7 大核心模块)

本项目基于 `Spring Cloud Alibaba` (Nacos) + `OpenFeign` + `RabbitMQ` 构建，采用严格的物理隔离。

| 微服务模块 | 端口 | 核心职责与架构意义 |
| :--- | :--- | :--- |
| `payment-api-gateway` | 8080 | **唯一入口**。基于 Spring Cloud Gateway，拦截所有外部请求，执行统一的 HMAC-SHA256 签名验签与 Nonce 防重放，实现内外网物理隔离。 |
| `payment-transaction-service`| 8081 | **交易大脑**。维护 10 种支付状态机，执行责任链风控，通过 Outbox 模式保证消息投递，绝不包含任何第三方 SDK 代码。 |
| `payment-channel-service` | 8085 | **渠道防腐层 (ACL)**。纯粹的底层协议转换器。通过 SPI/策略模式动态加载微信/支付宝 SDK，处理恶心的证书、加解密，并将原生回调转化为中台标准 JSON 放入 MQ。 |
| `payment-notification-service`| 8083 | **通知送达器**。专门消费交易成功的 MQ 消息，向商户发送 HTTP 回调。若失败则执行阶梯式指数退避重试（1m/5m/1h...）。 |
| `payment-reconciliation-service`| 8082 | **对账引擎**。T+1 定时任务，拉取第三方账单与本地数据库比对，识别长款、短款、金额不符，生成差异报告。 |
| `payment-admin-service` | 8089 | **运营 BFF**。面向内部运营/财务，提供商户 API-Key 动态生成、全局订单干预、手工强制退款等内网级管理接口。 |
| `payment-test-client` | 8088 | **模拟商户**。包含签名工具类和 Feign 拦截器，用于快速在本地模拟真实的电商下单闭环。 |

---

## 🧠 三、核心架构设计与难点攻克

本项目最出彩的地方在于对典型分布式难题的优雅解决：

### 1. 如何彻底解耦第三方支付渠道？（防腐层 + 策略模式）
*   **设计**：在 `payment-channel-service` 中，定义了统一的 `PayChannelStrategy` 接口。
*   **实现**：`WechatPayStrategy` 和 `AlipayStrategy` 分别实现该接口。利用 Spring 的 `@Component` 配合 `PayChannelFactory` 实现策略的动态路由。
*   **效果**：`transaction-service` 根本不知道微信和支付宝的存在，它只知道通过 Feign 调用 `channel-service` 获取支付参数。新增渠道（如 PayPal）只需加一个策略类，主干代码 0 侵入。

### 2. 如何保证支付状态不被并发覆盖？（状态机 + 乐观锁）
*   **痛点**：用户支付成功，此时网络卡顿，用户又点了一次取消订单。两个请求并发打到数据库，可能把“已支付”覆盖成“已取消”。
*   **设计**：定义了严格的有向无环状态机（`INIT` -> `PENDING` -> `PROCESSING` -> `SUCCESS/FAILED`）。
*   **实现**：在 MyBatis-Plus 的 Mapper 层，全部使用带预期状态的 Update 语句（乐观锁）。
    ```sql
    UPDATE payment_transaction 
    SET status = 'SUCCESS' 
    WHERE transaction_id = #{id} AND status = 'PROCESSING'
    ```
*   **效果**：只有在预期状态下更新才会成功（返回 affected rows > 0），彻底杜绝并发覆盖。

### 3. 如何保证商户 100% 收到通知？（Outbox 本地消息表模式）
*   **痛点**：交易服务更新了 MySQL 状态为 SUCCESS，接着向 RabbitMQ 发送通知消息时，网络断了或 MQ 宕机。MySQL 提交了，MQ 没发出去，导致商户永远收不到钱款到账的通知（双写不一致）。
*   **设计**：**抛弃 2PC 强一致事务，采用最终一致性**。
*   **实现**：
    1. 在 `payment_transaction_db` 中建立一张 `outbox_entity` 表。
    2. 在更新支付状态的同一个 `@Transactional` 中，向 `outbox_entity` 插入一条状态为 `PENDING` 的消息体。**（保证业务数据和消息同生共死）**
    3. 后台独立线程（`OutboxProcessor`）每隔 2 秒扫描 `PENDING` 状态的记录，投递给 RabbitMQ，收到 ACK 后标记为 `PROCESSED`。
*   **效果**：哪怕 MQ 宕机 3 天，只要恢复，消息依然会从数据库中被重新捞起投递，实现绝对的最终一致性。

### 4. 零侵入的风控引擎（责任链模式）
*   **设计**：在创建支付单的主流程前，挂载 `RiskControlManager`。
*   **实现**：实现了 `RiskRule` 接口的多个节点（黑名单 IP 拦截、高频防刷拦截、超大金额熔断）。通过链式调用 `chain.doFilter(context)` 进行前置阻断。

### 5. 坚若磐石的鉴权体系
*   **痛点**：防止中间人篡改支付金额、防止黑客截获请求后疯狂重放。
*   **实现**：
    *   **防篡改**：网关层要求 `X-Signature`。算法为 `HMAC-SHA256(Method + Path + Timestamp + Nonce + SHA256(Body), Secret)`。只要金额被改 1 分钱，签名校验立刻失败。
    *   **防重放**：网关层校验 `X-Nonce`，将其存入 Redis 并设置 5 分钟 TTL（配合 Timestamp 允许的最大误差）。同一个 Nonce 在 5 分钟内只能用一次。

---

## 💾 四、数据库物理隔离设计

项目根目录 `db/mysql/init/` 下包含了 5 个初始化脚本。它们被 Docker 挂载并在首次启动时自动执行，建立了 4 个完全独立的数据库：

1.  `payment_transaction_db`：包含流水表、退款表、回调审计表、本地消息表(Outbox)。
2.  `payment_notification_db`：包含异步通知记录表（记录重试次数与下次重试时间）。
3.  `payment_reconciliation_db`：包含对账批次表、长短款异常明细表。
4.  `payment_admin_db`：包含商户配置表（API Key / Secret / 渠道证书）。

---

## 🛠️ 五、系统收敛与高可用工程化 (System Convergence)

本项目不仅关注业务功能的实现，更注重系统在生产环境下的**可观测性**、**稳定性**与**容错能力**。

### 1. 全链路可观测性 (Observability)
*   **TraceId 穿透**：通过 `TraceContext` 和 SLF4J MDC，在 Gateway 处生成 `X-Trace-Id`。
*   **跨边界透传**：通过自定义 `FeignTraceInterceptor` 将 TraceId 放入 HTTP Header，通过重写 `RabbitTemplate` 和 `RabbitListenerContainerFactory` 将 TraceId 放入 MQ Header，实现了 HTTP 和 MQ 异步边界的完美透传。
*   **日志统一**：所有微服务共用 `logback-spring-base.xml`，日志输出强绑定 TraceId，彻底解决微服务排错“大海捞针”的问题。

### 2. 统一异常体系 (Unified Exception Handling)
*   **标准化错误码**：废弃了零散的 HTTP 状态码，引入全局 `ErrorCode` 枚举，明确区分系统异常（SYS_xxx）、业务异常（BIZ_xxx）与渠道异常（CHN_xxx），并标记 `retryable` 属性。
*   **全局拦截**：在 `payment-common` 提供 `@ControllerAdvice`，拦截所有微服务抛出的 `BaseException`，并自动装配 TraceId 组装为标准 JSON 响应给调用方。

### 3. Sentinel 熔断降级与限流 (High Availability)
*   **网关限流**：在 API Gateway 层接入 Sentinel，支持在 Dashboard 中动态配置 QPS 限流规则，保护内部服务不被突发流量击垮。
*   **RPC 熔断降级**：在 `transaction-service` 调用 `channel-service` 时接入 OpenFeign Fallback。当第三方渠道网络超时或大面积报错时，立刻触发熔断，返回自定义的降级响应，防止底层故障导致交易核心服务线程池耗尽（雪崩效应）。

---

## 💻 六、技术栈选型摘要

*   **Java 21**: 采用最新 LTS 版本。
*   **Spring Boot 3.2.3 + Spring Cloud 2023**: 现代微服务基础设施。
*   **Nacos**: 承担服务注册发现与配置中心。
*   **Sentinel**: 提供流量治理、限流防刷与熔断降级保护。
*   **MyBatis-Plus**: 摒弃繁琐的 XML，使用 Lambda 链式查询与面向对象的数据库操作。
*   **RabbitMQ**: 利用其高可靠的投递机制与死信队列（DLQ）特性，完成回调削峰与通知重试。
*   **Redis**: 承担网关防重放（SETNX）、并发控制分布式锁、以及风控规则的高速缓存。
*   **Docker Compose**: 实现了整个庞大架构的 1 键本地拉起。

---

> **给 AI 助手的提示**：
> 当你接手或协助开发本项目时，请严格遵守“渠道防腐”、“状态机控制”、“最终一致性投递”的既定架构原则。任何越过网关的直接访问、任何破坏 Outbox 事务完整性的行为都是被绝对禁止的。
