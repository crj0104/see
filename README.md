# Distributed Payment Core Platform
> 企业级分布式插件化支付中台

本项目是一个基于 Spring Cloud Alibaba 体系构建的企业级支付中台。它屏蔽了底层不同第三方支付渠道的复杂性，向上层业务系统提供统一的支付、退款、回调与对账能力。

## 1. 系统架构

项目采用微服务架构，包含以下 7 个核心模块：
*   **payment-common**：公共核心包（实体、DTO、异常、工具类、基础配置）。
*   **payment-api-gateway**：全局 API 网关（请求路由、HMAC-SHA256 签名鉴权、防重放拦截）。
*   **payment-transaction-service**：核心交易大脑（支付下单、退款、状态机控制、风控拦截、Outbox 消息持久化）。
*   **payment-channel-service**：渠道适配服务（防腐层，隔离微信/支付宝 SDK，处理底层协议、加解密与原生回调）。
*   **payment-notification-service**：异步通知服务（支付结果回调业务方、阶梯重试补偿）。
*   **payment-reconciliation-service**：对账服务（T+1 第三方账单核对与差异处理）。
*   **payment-admin-service**：运营管理后台（BFF 层，面向内部员工提供商户密钥管理、交易查询、手工平账等接口）。
*   **payment-test-client**：模拟商户端（自带签名逻辑，用于快速验证电商下单与支付闭环）。

### 核心技术栈
*   **框架**: Spring Boot 3.2.x, Spring Cloud 2023.x, Spring Cloud Alibaba (Nacos)
*   **数据库**: MySQL 8.0+ (MyBatis-Plus)
*   **缓存与分布式锁**: Redis 7.0+ (Lettuce)
*   **通信与消息**: OpenFeign (同步) + RabbitMQ (异步)
*   **鉴权体系**: API Key + HMAC-SHA256 动态签名 + Nonce 防重放
*   **架构模式**: 渠道防腐层(ACL)、Outbox本地消息表(最终一致性)
*   **设计模式**: 责任链模式(风控)、策略模式+工厂模式(渠道插件化)、状态机模式(订单流转)

---

## 2. Docker Compose 一键部署 (推荐)

本项目提供了完整的企业级一键拉起脚本。

1. 确保已安装 Docker 和 Docker Compose。
2. 在项目根目录执行：
   ```bash
   docker-compose up -d
   ```
3. 等待约 30 秒。MySQL 容器启动时会自动执行 `db/mysql/init/` 下的脚本，完成 **4个微服务专属库** 和 **8张核心表** 的创建与测试数据初始化。
4. 验证基础设施状态：
   * Nacos 控制台: `http://localhost:8848/nacos` (nacos/nacos)
   * RabbitMQ 控制台: `http://localhost:15672` (guest/guest)

---

## 3. 微服务启动顺序

当基础设施就绪后，在 IDE 中按以下顺序启动 Spring Boot 应用：

1. `PaymentGatewayApplication` (网关, 8080)
2. `PaymentTransactionApplication` (交易核心, 8081)
3. `PaymentChannelApplication` (渠道防腐, 8085)
4. `PaymentNotificationApplication` (商户通知, 8083)
5. `PaymentReconciliationApplication` (对账服务, 8082)
6. `PaymentAdminApplication` (运营后台, 8089)
7. `PaymentTestClientApplication` (模拟测试商城, 8088)

---

## 4. 快速体验支付闭环

所有服务启动后，打开 Postman 或浏览器发起请求，模拟电商业务调用：

**1. 模拟电商触发支付购买:**
```http
POST http://localhost:8088/mock/mall/buy?productName=MacBook&price=9999&paymentMethod=WECHAT_PAY
```
> `payment-test-client` 会自动完成 HMAC-SHA256 签名，穿透网关，生成 `transactionId` 并返回。

**2. 模拟收银台点击确认:**
```http
POST http://localhost:8088/mock/mall/pay/{第一步返回的transactionId}
```
> 交易服务将流转状态机，通过 Feign 调用渠道服务获取第三方拉起参数。

**3. 体验异步回调与商户通知:**
向网关发送原生回调（模拟微信服务器）：
```http
POST http://localhost:8080/api/v1/callbacks/provider/wechat_pay
Content-Type: application/json

{
    "transactionId": "{你的transactionId}",
    "providerTransactionId": "WX123456789"
}
```
> 网关 -> 渠道服务验签 -> RabbitMQ -> 交易服务修改状态 -> Outbox持久化 -> 通知服务 -> 测试商城的 MockCallbackController 打印发货日志。

---

## 5. 项目亮点与企业级特性

*   **真正的零 `if-else` 渠道路由**：基于 SPI 与 Spring 自动装配，新增渠道只需新建一个类实现 `PayChannelStrategy` 接口，主干代码零侵入。
*   **渠道防腐层 (ACL)**：彻底抛弃“在一个系统里集成所有 SDK”的做法。底层加密和网络请求被物理隔离在 `payment-channel-service`，避免 Jar 包地狱。
*   **严密的资金防线**：
    *   防瞬时并发：Redis `SETNX` 拦截 + MySQL `UNIQUE KEY` 兜底。
    *   防状态覆盖：MyBatis Mapper 级别实现基于预期状态的乐观锁（`UPDATE ... WHERE status = expectStatus`）。
*   **100% 消息最终一致性**：彻底抛弃 2PC 强一致分布式事务，采用 `Outbox (本地消息表)` 模式，确保业务入库与消息持久化在同一事务中完成，后台 Job 异步投递 MQ，附带死信队列补偿。
*   **前置风控拦截**：基于责任链模式（`RiskRule`），在订单落库前进行 IP 黑名单、高频拦截与大额熔断。
