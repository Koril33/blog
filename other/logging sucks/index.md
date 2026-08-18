---
title: "logging sucks"
date: 2026-08-12T11:51:36
summary: "译文：日志糟透了"
---

## 原文

https://loggingsucks.com/

---

## 前言

你的日志在欺骗你，不过并非恶意性的，而是它们根本无法提供真相。

你可能已经花费了大量的时间在日志里检索（grep），试图弄明白为什么用户无法结账，为什么 webhook 失败了，为什么你的 P99 在凌晨 3 点突然飙升。结果你一无所获，只有一些没用的时间戳和模糊的信息，仿佛在嘲笑着你。

这并不是你的错。目前常用的日志记录方式存在根本性的缺陷。而且，在你的代码库里添加 OpenTelemetry 并不能神奇地解决问题。

让我来告诉你问题出在哪里，更重要的是，如何解决它。

---

## 核心问题

日志，曾是为了另一个时代而设计的。属于单体架构、单服务器的时代，问题都可以在本地复现。而如今一个用户请求可能涉及 15 个服务，3 个数据库，2 个缓存和一个消息队列。你的日志却仍然停留在 2005 年的水平。

以下是一个典型的日志配置示例：

一个用户可能每秒产生 17 行日志：

```
10:23:45.612 INFO [gateway] Response sent: 200 OK (489ms)
10:23:45.589 INFO [orders] Order confirmed: order_789
10:23:45.567 INFO [payments] Payment successful, charge_ch_abc
10:23:45.534 INFO [payments] Retrying payment, attempt 2
10:23:45.502 ERROR [payments] Payment declined: insufficient_funds
10:23:45.456 WARN [payments] Stripe API slow response: 145ms
10:23:45.401 DEBUG [payments] Calling Stripe API, attempt 1
10:23:45.367 INFO [payments] Processing payment via Stripe
10:23:45.334 DEBUG [inventory] Item SKU-123 in stock: 47 units
10:23:45.310 WARN [inventory] Low stock warning: SKU-456 has only 3 units left
10:23:45.298 INFO [inventory] Checking stock for SKU-123
10:23:45.267 DEBUG [orders] Validating 3 items in cart
10:23:45.234 INFO [orders] Creating order for cart_xyz
10:23:45.201 DEBUG [auth] Token valid, expires in 3600s
10:23:45.189 INFO [auth] Validating JWT token for user
10:23:45.156 DEBUG [gateway] Parsing request body, size: 2.4KB
10:23:45.123 INFO [gateway] Received request POST /api/checkout 
```

三个并发用户访问会产生 51 条日志：

```
 10:23:47.245 INFO [orders] Order confirmed: order_789
10:23:47.221 INFO [payments] Payment successful, charge_ch_abc
10:23:47.205 INFO [gateway] Response sent: 200 OK (489ms)
10:23:47.130 INFO [payments] Retrying payment, attempt 2
10:23:47.101 ERROR [payments] Payment declined: insufficient_funds
10:23:47.045 WARN [payments] Stripe API slow response: 145ms
10:23:47.039 DEBUG [payments] Calling Stripe API, attempt 1
10:23:46.961 INFO [payments] Processing payment via Stripe
10:23:46.952 WARN [inventory] Low stock warning: SKU-456 has only 3 units left
10:23:47.093 INFO [gateway] Response sent: 200 OK (489ms)
10:23:46.929 DEBUG [inventory] Item SKU-123 in stock: 47 units
10:23:46.923 INFO [inventory] Checking stock for SKU-123
10:23:46.870 DEBUG [orders] Validating 3 items in cart
10:23:46.836 INFO [orders] Creating order for cart_xyz
10:23:46.998 INFO [payments] Payment successful, charge_ch_abc
10:23:46.998 INFO [orders] Order confirmed: order_789
10:23:46.989 INFO [payments] Retrying payment, attempt 2
10:23:46.969 ERROR [payments] Payment declined: insufficient_funds
10:23:46.798 DEBUG [auth] Token valid, expires in 3600s
10:23:46.791 INFO [auth] Validating JWT token for user
10:23:46.731 DEBUG [gateway] Parsing request body, size: 2.4KB
10:23:46.874 WARN [payments] Stripe API slow response: 145ms
10:23:46.702 INFO [gateway] Received request POST /api/checkout
10:23:46.832 DEBUG [inventory] Item SKU-123 in stock: 47 units
10:23:46.830 INFO [payments] Processing payment via Stripe
10:23:46.811 DEBUG [payments] Calling Stripe API, attempt 1
10:23:46.763 INFO [inventory] Checking stock for SKU-123
10:23:46.717 INFO [orders] Creating order for cart_xyz
10:23:46.714 WARN [inventory] Low stock warning: SKU-456 has only 3 units left
10:23:46.709 DEBUG [orders] Validating 3 items in cart
10:23:46.659 INFO [auth] Validating JWT token for user
10:23:46.647 DEBUG [gateway] Parsing request body, size: 2.4KB
10:23:46.611 DEBUG [auth] Token valid, expires in 3600s
10:23:46.592 INFO [gateway] Received request POST /api/checkout
10:23:46.292 INFO [orders] Order confirmed: order_789
10:23:46.245 INFO [payments] Payment successful, charge_ch_abc
10:23:46.223 INFO [gateway] Response sent: 200 OK (489ms)
10:23:46.187 INFO [payments] Retrying payment, attempt 2
10:23:46.144 WARN [payments] Stripe API slow response: 145ms
10:23:46.130 ERROR [payments] Payment declined: insufficient_funds
10:23:46.056 INFO [payments] Processing payment via Stripe
10:23:46.046 DEBUG [payments] Calling Stripe API, attempt 1
10:23:46.028 DEBUG [inventory] Item SKU-123 in stock: 47 units
10:23:46.000 WARN [inventory] Low stock warning: SKU-456 has only 3 units left
10:23:45.966 DEBUG [orders] Validating 3 items in cart
10:23:45.928 INFO [inventory] Checking stock for SKU-123
10:23:45.903 DEBUG [auth] Token valid, expires in 3600s
10:23:45.856 DEBUG [gateway] Parsing request body, size: 2.4KB
10:23:45.846 INFO [orders] Creating order for cart_xyz
10:23:45.812 INFO [auth] Validating JWT token for user
10:23:45.808 INFO [gateway] Received request POST /api/checkout 
```

译注：作者这里用一个很直观的前端控件来展示 1-30 个并发用户下的日志变化，建议访问原网站使用。

现在想象一下，有 10000 个用户同时这样做。祝你好运，看看能不能找到任何东西。

一次成功的请求会产生 17 行日志。现在乘以 10,000 个并发用户，每秒就会产生 130,000 行日志。其中大部分日志毫无用处。

但真正的问题在于：当出现问题时，这些日志帮不了你。它们缺少你最需要的东西：上下文（context）。

---

## 为什么字符串搜索功能失效了

当用户报告“我无法完成购买”时，你的第一反应是搜索日志。你输入他们的邮箱地址，或者可能是他们的用户 ID，然后按下回车键。

### 徒劳的搜索

搜索“user-123”并找到日志行，但请注意你缺少多少上下文信息。错误发生了，但为什么会发生？

示例日志如下：

```
[INFO] 2025-01-15 10:23:41 Server started on port 3000
[DEBUG] 2025-01-15 10:23:42 Database connection pool initialized
[INFO] 2025-01-15 10:23:45 Processing request for user user-123
[DEBUG] 2025-01-15 10:23:45 Cache lookup: session_abc123
[INFO] 2025-01-15 10:23:46 Request received from 192.168.1.50
[DEBUG] 2025-01-15 10:23:46 Parsing JSON body, size: 1.2KB
[INFO] 2025-01-15 10:23:47 user_id=user-123 action=checkout started
[DEBUG] 2025-01-15 10:23:47 Loading cart items from database
[INFO] 2025-01-15 10:23:48 Request from user-1234 processed
[DEBUG] 2025-01-15 10:23:48 Validating payment method
[WARN] 2025-01-15 10:23:49 Slow query detected: 450ms
[INFO] 2025-01-15 10:23:49 {"user":"user-123","event":"payment_failed"}
[DEBUG] 2025-01-15 10:23:50 Retrying payment, attempt 2
[ERROR] 2025-01-15 10:23:50 Payment gateway timeout
[INFO] 2025-01-15 10:23:51 UserID: user-123 - Session timeout warning
[DEBUG] 2025-01-15 10:23:51 Refreshing session token
[INFO] 2025-01-15 10:23:52 Request completed for user-12345
[DEBUG] 2025-01-15 10:23:52 Cleaning up temporary files
[INFO] 2025-01-15 10:23:53 [user-123] Cart updated, 3 items
[DEBUG] 2025-01-15 10:23:53 Broadcasting cart update event
[INFO] 2025-01-15 10:23:54 Health check passed
[DEBUG] 2025-01-15 10:23:54 Memory usage: 45%
[INFO] 2025-01-15 10:23:55 Background job started: email-queue
[DEBUG] 2025-01-15 10:23:55 Processing 12 pending emails
[WARN] 2025-01-15 10:23:56 Rate limit approaching for IP 192.168.1.50
[INFO] 2025-01-15 10:23:56 WebSocket connection established
[DEBUG] 2025-01-15 10:23:57 Subscribing to channels: [orders, notifications]
[INFO] 2025-01-15 10:23:57 Order #4521 created successfully
[DEBUG] 2025-01-15 10:23:58 Sending order confirmation email
[INFO] 2025-01-15 10:23:58 Inventory updated for SKU-789 
```

grep "user-123" 的结果：

```
[INFO] 2025-01-15 10:23:45 Processing request for user user-123
[INFO] 2025-01-15 10:23:47 user_id=user-123 action=checkout started
[INFO] 2025-01-15 10:23:48 Request from user-1234 processed
[INFO] 2025-01-15 10:23:49 {"user":"user-123","event":"payment_failed"}
[INFO] 2025-01-15 10:23:51 UserID: user-123 - Session timeout warning
[INFO] 2025-01-15 10:23:52 Request completed for user-12345
[INFO] 2025-01-15 10:23:53 [user-123] Cart updated, 3 items
```

注意到什么了吗？同一个用户 ID 有 5 种不同的格式：

```
user user-123
user_id=user-123
{"user":"user-123"}
UserID: user-123
[user-123]
```

字符串搜索把日志看作是一连串的字符。它无法理解结构，没有关系的概念，也无法关联跨服务的事件。

当你搜索“user-123”时，你可能会发现它在代码库中以 47 种不同的方式被记录：

```
user-123
user_id=user-123
{"userId": "user-123"}
[USER:user-123]
processing user: user-123
```

而这些仅仅是包含用户 ID 的日志。那么下游服务只记录了订单 ID 呢？现在你需要进行第二次搜索。然后是第三次。你就像被束缚了一只手的侦探。

根本问题在于：日志是针对写入而优化的，而不是针对查询的。

开发者写 console.log("支付失败") 是因为这样操作起来方便快捷。他们根本不会想到，在凌晨两点系统宕机的时候，会有人苦苦寻找这个信息。

---

## 定义一些术语

在向你展示解决方法之前，让我先解释几个术语。这些术语经常被滥用，而且常常被错误使用。

结构化日志（Structured Logging）：日志以键值对（通常为 JSON 格式）而非纯字符串的形式输出。例如，使用 `{"event": "payment_failed", "user_id": "123"}` 而不是 `"Payment failed for user 123".`。结构化日志是必要的，但并非充分条件。

基数（Cardinality）：字段可以包含的唯一值的数量。user_id 字段的基数很高（数百万个唯一值）。http_method 字段的基数很低（GET、POST、PUT、DELETE 等）。正是高基数字段使得日志在调试中真正有用。

维度（Dimensionality）：日志事件中的字段数量。包含 5 个字段的日志维度较低。包含 50 个字段的日志维度较高。维度越多，可以回答的问题就越多。

宽事件（Wide Event）：每个请求、每个服务都会发出一个包含丰富上下文信息的单一日志事件。这样，你就无需为每个请求生成 13 行日志，只需生成一行包含 50 多个字段的日志，其中包含调试所需的一切信息。

规范日志行（Canonical Log Line，或者叫权威日志行）：这是“宽事件”的另一种说法，由 Stripe 推广开来。每个请求对应一条日志行，作为事件发生的权威记录。

高基数字段（例如 user_id）对调试最有价值。讽刺的是：大多数日志系统按日志量收费，却无法处理高基数字段。这完全本末倒置。高基数恰恰是调试所需要的。

---

## OpenTelemetry 也救不了你

我经常看到这样的说法：“只要用 OpenTelemetry，你的可观测性问题就迎刃而解了。”

并非如此。OpenTelemetry 是一种协议和一套 SDK。它规范了遥测数据（日志、追踪、指标）的收集和导出方式。这确实很有用：这意味着你不会被限制在特定供应商的格式中。

但 OpenTelemetry 无法做到以下几点：

1. 它不会决定记录什么。你仍然需要有意识地对代码进行插桩。

2. 它不会添加业务上下文。如果你不添加用户的订阅级别、购物车金额或启用的功能标志，OpenTelemetry 不会自动识别。

3. 它不会改变你的思维模式。如果你仍然以“日志语句”的视角思考问题，你只会以标准化的格式输出错误的遥测数据。

OpenTelemetry 是一种交付机制。它并不知道 user-789 是你三年的老客户，并且刚刚尝试消费 160 美元。你需要告诉它这一点。

---

## 解决方案：宽事件/规范日志行

这种思维模式的转变将改变一切：不要记录代码正在执行的操作，而是记录此请求发生了什么。

不要再把日志当作调试日志，而应该把它看作是结构化的业务事件记录。

对于每个请求，每次服务跳转都应该发出一个宽事件。这个事件应该包含所有可能有助于调试的上下文信息，不仅包括出错的原因，还要包含请求的完整过程。

仅包含 5 个字段的格式化日志：

```json
{
  "request_id": "req_8bf7ec2d",
  "method": "POST",
  "path": "/api/checkout",
  "status_code": 500,
  "duration_ms": 1247
}
```

包含 72 个字段的高维度宽事件：

```json
{
  "request_id": "req_8bf7ec2d",
  "trace_id": "abc123def456",
  "span_id": "span_789",
  "parent_span_id": "span_456",
  "method": "POST",
  "path": "/api/checkout",
  "query_params": {
    "utm_source": "email"
  },
  "status_code": 500,
  "duration_ms": 1247,
  "timestamp": "2025-01-15T10:23:45.612Z",
  "client_ip": "192.168.1.42",
  "user_agent": "Mozilla/5.0...",
  "content_type": "application/json",
  "request_size_bytes": 2048,
  "response_size_bytes": 512,
  "user_id": "user_456",
  "session_id": "sess_abc123",
  "subscription_tier": "premium",
  "account_age_days": 847,
  "lifetime_value_cents": 284700,
  "organization_id": "org_acme",
  "team_id": "team_engineering",
  "role": "admin",
  "country": "US",
  "locale": "en-US",
  "timezone": "America/New_York",
  "feature_flags": {
    "new_checkout_flow": true,
    "beta_features": false
  },
  "ab_test_cohort": "experiment_a",
  "order_id": "order_789",
  "cart_id": "cart_xyz",
  "cart_total_cents": 15999,
  "item_count": 3,
  "item_skus": [
    "SKU-001",
    "SKU-002",
    "SKU-003"
  ],
  "payment_method": "card",
  "payment_provider": "stripe",
  "payment_intent_id": "pi_abc123",
  "coupon_code": "SAVE20",
  "discount_cents": 3200,
  "shipping_method": "express",
  "shipping_cents": 999,
  "currency": "USD",
  "is_gift": false,
  "service_name": "checkout-service",
  "service_version": "2.4.1",
  "deployment_id": "deploy_789",
  "git_sha": "a1b2c3d",
  "region": "us-east-1",
  "availability_zone": "us-east-1a",
  "host": "checkout-5f8d9b7c6d-abc12",
  "container_id": "ctr_xyz789",
  "k8s_namespace": "production",
  "k8s_pod": "checkout-5f8d9b7c6d-abc12",
  "cloud_provider": "aws",
  "environment": "production",
  "error_type": "PaymentError",
  "error_code": "card_declined",
  "error_message": "Card declined by issuer",
  "error_retriable": false,
  "error_stack": "PaymentError: Card declined...",
  "stripe_decline_code": "insufficient_funds",
  "retry_count": 3,
  "last_retry_at": "2025-01-15T10:23:44.000Z",
  "upstream_service": "stripe-api",
  "upstream_latency_ms": 1089,
  "db_query_count": 12,
  "db_query_time_ms": 156,
  "cache_hits": 8,
  "cache_misses": 2,
  "external_call_count": 3,
  "external_call_time_ms": 890,
  "memory_used_mb": 128,
  "cpu_time_ms": 45
}
```

通过记录的事件，试试能否回答以下问题？

- 为什么用户 X 的结账失败了？
- 高级用户是否遇到更多错误？
- 是哪个部署导致了延迟回升？
- 新结账功能的错误率是多少？

在实践中，一个宽事件的例子：

```json
{
  "timestamp": "2025-01-15T10:23:45.612Z",
  "request_id": "req_8bf7ec2d",
  "trace_id": "abc123",

  "service": "checkout-service",
  "version": "2.4.1",
  "deployment_id": "deploy_789",
  "region": "us-east-1",

  "method": "POST",
  "path": "/api/checkout",
  "status_code": 500,
  "duration_ms": 1247,

  "user": {
    "id": "user_456",
    "subscription": "premium",
    "account_age_days": 847,
    "lifetime_value_cents": 284700
  },

  "cart": {
    "id": "cart_xyz",
    "item_count": 3,
    "total_cents": 15999,
    "coupon_applied": "SAVE20"
  },

  "payment": {
    "method": "card",
    "provider": "stripe",
    "latency_ms": 1089,
    "attempt": 3
  },

  "error": {
    "type": "PaymentError",
    "code": "card_declined",
    "message": "Card declined by issuer",
    "retriable": false,
    "stripe_decline_code": "insufficient_funds"
  },

  "feature_flags": {
    "new_checkout_flow": true,
    "express_payment": false
  }
}
```

一个事件，满足你所有需求。当该用户投诉时，你搜索 user_id = "user_456"，即可立即获知：

- 他们是高级客户（优先级高）
- 他们已与你合作超过两年（优先级非常高）
- 第三次付款尝试失败
- 实际原因是：余额不足
- 他们使用的是新的结账流程（可能存在关联？）

无需使用 grep 命令，无需猜测，无需进行二次搜索。

---

## 你现在可以运行的查询

使用宽事件，你不再搜索文本，而是查询结构化数据。两者之间的区别可谓天壤之别。

这就是宽事件与高基数、高维度数据相结合的强大之处。你不再需要搜索日志，而是可以直接对生产流量进行分析。

---

## 实现宽事件

这里提供一种实用的实现模式。关键在于：在整个请求生命周期中构建事件，然后在请求结束时一次性发出。

```javascript
// middleware/wideEvent.ts
export function wideEventMiddleware() {
  return async (ctx, next) => {
    const startTime = Date.now();

    // Initialize the wide event with request context
    const event: Record<string, unknown> = {
      request_id: ctx.get('requestId'),
      timestamp: new Date().toISOString(),
      method: ctx.req.method,
      path: ctx.req.path,
      service: process.env.SERVICE_NAME,
      version: process.env.SERVICE_VERSION,
      deployment_id: process.env.DEPLOYMENT_ID,
      region: process.env.REGION,
    };

    // Make the event accessible to handlers
    ctx.set('wideEvent', event);

    try {
      await next();
      event.status_code = ctx.res.status;
      event.outcome = 'success';
    } catch (error) {
      event.status_code = 500;
      event.outcome = 'error';
      event.error = {
        type: error.name,
        message: error.message,
        code: error.code,
        retriable: error.retriable ?? false,
      };
      throw error;
    } finally {
      event.duration_ms = Date.now() - startTime;

      // Emit the wide event
      logger.info(event);
    }
  };
}
```

然后，在你的处理程序中，你可以使用业务上下文来丰富事件：

```javascript
app.post('/checkout', async (ctx) => {
  const event = ctx.get('wideEvent');
  const user = ctx.get('user');

  // Add user context
  event.user = {
    id: user.id,
    subscription: user.subscription,
    account_age_days: daysSince(user.createdAt),
    lifetime_value_cents: user.ltv,
  };

  // Add business context as you process
  const cart = await getCart(user.id);
  event.cart = {
    id: cart.id,
    item_count: cart.items.length,
    total_cents: cart.total,
    coupon_applied: cart.coupon?.code,
  };

  // Process payment
  const paymentStart = Date.now();
  const payment = await processPayment(cart, user);

  event.payment = {
    method: payment.method,
    provider: payment.provider,
    latency_ms: Date.now() - paymentStart,
    attempt: payment.attemptNumber,
  };

  // If payment fails, add error details
  if (payment.error) {
    event.error = {
      type: 'PaymentError',
      code: payment.error.code,
      stripe_decline_code: payment.error.declineCode,
    };
  }

  return ctx.json({ orderId: payment.orderId });
});
```

json 随着业务推进，变化如下：

收到请求
使用请求上下文初始化事件

```json
{
  "request_id": "req_8bf7ec2d",
  "timestamp": "2025-01-15T10:23:45.612Z",
  "method": "POST",
  "path": "/api/checkout",
  "service": "checkout-service"
}
```

用户已通过身份验证
从身份验证中间件添加用户上下文

```json
{
  "user": {
    "id": "user_456",
    "subscription": "premium",
    "account_age_days": 847,
    "lifetime_value_cents": 284700
  },
  "request_id": "req_8bf7ec2d",
  "timestamp": "2025-01-15T10:23:45.612Z",
  "method": "POST",
  "path": "/api/checkout",
  "service": "checkout-service"
}
```

购物车已加载
从购物车服务添加业务上下文

```json
{
  "cart": {
    "id": "cart_xyz",
    "item_count": 3,
    "total_cents": 15999,
    "coupon_applied": "SAVE20"
  },
  "user": {
    "id": "user_456",
    "subscription": "premium",
    "account_age_days": 847,
    "lifetime_value_cents": 284700
  },
  "request_id": "req_8bf7ec2d",
  "timestamp": "2025-01-15T10:23:45.612Z",
  "method": "POST",
  "path": "/api/checkout",
  "service": "checkout-service"
}
```

支付处理
开始支付并记录时间

```json
{
  "payment": {
    "method": "card",
    "provider": "stripe"
  },
  "cart": {
    "id": "cart_xyz",
    "item_count": 3,
    "total_cents": 15999,
    "coupon_applied": "SAVE20"
  },
  "user": {
    "id": "user_456",
    "subscription": "premium",
    "account_age_days": 847,
    "lifetime_value_cents": 284700
  },
  "request_id": "req_8bf7ec2d",
  "timestamp": "2025-01-15T10:23:45.612Z",
  "method": "POST",
  "path": "/api/checkout",
  "service": "checkout-service"
}
```

付款失败
记录错误详情及上下文信息

```json
{
  "payment": {
    "method": "card",
    "provider": "stripe"
  },
  "error": {
    "type": "PaymentError",
    "code": "card_declined",
    "message": "Card declined by issuer",
    "stripe_decline_code": "insufficient_funds"
  },
  "cart": {
    "id": "cart_xyz",
    "item_count": 3,
    "total_cents": 15999,
    "coupon_applied": "SAVE20"
  },
  "user": {
    "id": "user_456",
    "subscription": "premium",
    "account_age_days": 847,
    "lifetime_value_cents": 284700
  },
  "request_id": "req_8bf7ec2d",
  "timestamp": "2025-01-15T10:23:45.612Z",
  "method": "POST",
  "path": "/api/checkout",
  "service": "checkout-service"
}
```

事件已发出
最终确定持续时间和状态，并发送至日志记录器。

```json
{
  "duration_ms": 1247,
  "status_code": 500,
  "outcome": "error",
  "payment": {
    "method": "card",
    "provider": "stripe"
  },
  "error": {
    "type": "PaymentError",
    "code": "card_declined",
    "message": "Card declined by issuer",
    "stripe_decline_code": "insufficient_funds"
  },
  "cart": {
    "id": "cart_xyz",
    "item_count": 3,
    "total_cents": 15999,
    "coupon_applied": "SAVE20"
  },
  "user": {
    "id": "user_456",
    "subscription": "premium",
    "account_age_days": 847,
    "lifetime_value_cents": 284700
  },
  "request_id": "req_8bf7ec2d",
  "timestamp": "2025-01-15T10:23:45.612Z",
  "method": "POST",
  "path": "/api/checkout",
  "service": "checkout-service"
}
```

---

## 抽样：控制成本

“但是 Boris，”我仿佛听到你在说，“如果我每秒处理 10,000 个请求，每个请求记录 50 个字段，我的可观测性费用会让我破产。”

这确实是个值得担忧的问题。这就是抽样的作用所在。

抽样意味着只保留一部分事件。与其存储 100% 的流量，不如只存储 10% 或 1%。在大规模应用中，这是保持理智（和偿付能力）的唯一方法。

但是，简单的抽样是危险的。如果你随机抽取 1% 的流量，你可能会不小心丢失导致故障的那个请求。

### 尾部采样

尾部采样是指在请求完成后，根据其结果来决定是否进行采样。

规则很简单：

1. 始终保留错误请求。所有 500 错误、异常和失败请求都会被存储。

2. 始终保留慢请求。任何延迟超过 p99 阈值的请求都会被存储。

3. 始终保留特定用户请求。例如 VIP 客户、内部测试账号和已标记的会话。

4. 对剩余请求进行随机采样。对于响应迅速且响应正常的请求，保留 1-5% 的请求。

这样一来，你就能兼得两全：既能控制成本，又不会错过任何重要的事件。

```javascript
// Tail sampling decision function
function shouldSample(event: WideEvent): boolean {
  // Always keep errors
  if (event.status_code >= 500) return true;
  if (event.error) return true;

  // Always keep slow requests (above p99)
  if (event.duration_ms > 2000) return true;

  // Always keep VIP users
  if (event.user?.subscription === 'enterprise') return true;

  // Always keep requests with specific feature flags (debugging rollouts)
  if (event.feature_flags?.new_checkout_flow) return true;

  // Random sample the rest at 5%
  return Math.random() < 0.05;
}
```

---

## 误解

### 结构化日志和宽事件是一样的

不。结构化日志指的是你的日志是 JSON 格式而不是字符串格式。这是基本要求。宽事件是一种理念：每个请求对应一个包含所有上下文的完整事件。即使你拥有结构化的日志，也可能毫无用处（5 个字段，没有用户上下文，分散在 20 行日志中）。

### 我们已经在使用 OpenTelemetry 了，所以没问题

你使用的是交付机制。OpenTelemetry 不会决定捕获哪些信息，而是由你来决定。我见过的大多数 OTel 实现都只捕获最基本的信息：span 名称、持续时间和状态。这远远不够。你需要有意识地添加业务上下文信息。

### 这只是多了些步骤的追踪

追踪可以让你了解请求在各个服务之间的流转（哪个服务调用了哪个服务）。宽事件则可以让你了解服务内部的上下文。它们是互补的。理想情况下，你的宽事件就是你的追踪 span，并添加了所有你需要的上下文信息。

### 日志用于调试，指标用于仪表盘

这种区分是人为的，而且有害。宽泛的事件可以同时支持两者。查询宽泛的事件用于调试，聚合宽泛的事件用于仪表盘。数据本身是相同的，只是呈现方式不同。

### 高基数数据成本高昂且速度慢

对于为低基数字符串搜索而构建的传统日志系统来说，高基数数据确实成本高昂。现代列式数据库（例如 ClickHouse、BigQuery 等）专为高基数、高维数据而设计。工具已经跟上了时代的步伐，你的实践也应该随之改变。

---

## 回报

正确实施宽事件后，调试将不再是考古挖掘，而是数据分析。

不再是：“用户说结账失败。让我搜索 50 个服务，看看能不能找到什么线索。”

而是：“显示过去一小时内启用新结账流程的高级用户的所有结账失败记录，并按错误代码分组。”

一次查询，不到一秒即可获得结果，并迅速找到根本原因。

你的日志不再欺骗你，而是开始讲述真相。全部真相。

---

## 将这些实践应用到你的代码中

我已将本文中的关键知识点提炼成一项针对智能体编码的技能。它将帮助你在代码库中实现宽事件、高基数和高维度的日志记录。

```
npx add-skill boristane/agent-skills --skill "logging-best-practices"
```

我还在开发 polylane.com，因为在 2026 年，没有人应该再值班了。

---

## 译后记

这篇文章清晰的阐释了传统日志为什么在服务异常的时候，无法给予工程师有用的帮助，甚至是在拖后腿。

从人和项目的角度来说，日志的无能其中一个原因就是设计、开发阶段的偷懒和赶工。写一行 log.info("task execute something") 远远比思考这个任务从开始到结束，运行时可能发生的异常，精心构造上下文供后续分析这些事情，来的省力得多。

系统的运行是一个连续的过程，在存储空间，记录信息这两件事情上，我们做了取舍，把连续的运行离散成了一行行的字符串，问题在于信息的丢失，传统的行记录日志并不适合存在并发任务的系统，更别提分布式的应用，在这个过程中，行记录日志丢失了大量的上下文信息，这些上下文是后续分析和排查故障的关键，所以就会导致一个很尴尬的局面：开发人员记录了日志，这些日志实打实的占用了服务器的磁盘空间，但是日志信息过于分散，无法提供任何有帮助的信息，甚至会产生误导。

我觉得关键性的思维转变在于，要在有限离散的日志信息中，保存好上下文信息，并且把日志的角色从无意义的记录器，转变成能够回答工程师问题的可靠信息来源。

原始日志的排查故障就是一种未知的探索，因为往往这些日志未经过精心思考，写下后，就再也不会去看了，直到一个凌晨 2 点的告警或者用户投诉，运维人员进到服务器里执行`grep "user-id=123" ./error.log`，才会捶胸顿足，当时为什么没能让这行日志多保存一些信息。

开发或者运维人员扮演的角色应该是工程师而不是探险家或者侦探。当我们需要绞尽脑汁才能推断出某个异常的原因的时候，首先不是要去夸赞某个灵机一动的人像福尔摩斯一样明察秋毫，而是思考一下为什么排查故障如此艰难，明明设计、开发阶段我们本可以提前存储一些有用的信息。

让日志告诉我们信息，而不是和日志信息玩猜谜，这是工程化的一个进步。

在 AI 普及的时代，也许我们能做的就是在离散日志里尽可能的权衡好存储成本与信息量，设计好日志结构和流程，多存储一些有用的关键信息，AI 拿到这些有着丰富上下文信息的时候，会告诉我们真正的问题。
