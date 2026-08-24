---
title: "日志的结构化和分析"
date: 2026-08-18T11:09:40
summary: "对于 logging sucks 文章思路的讨论和测试"
---

## 前言

传统日志设计，使用的是无结构的文本格式，这在简单的项目中表现良好，并且易于实现，但并不适用于现代更加复杂的项目，尤其是存在大量并发，长流程的场景下，查看日志来排错变成了近乎折磨的任务。

结构化日志和构建丰富的上下文场景信息就是一个初步的解法，本文从一个随机生成日志的脚本触发，来对于传统日志和结构化日志的区别，以及如何分析数据得到有用的信息。

---

## 传统日志

以一个用户从登录到支付的流程为例：用户输入用户名密码，点击登录按钮，调用登录接口，成功后，用户点击支付按钮，输入密码，调用支付接口，登录和支付都可能因为各种原因失败。

传统的日志如下：

```
[2026-08-18 11:25:28,866] - INFO - 调用 login api, username='alice'
[2026-08-18 11:25:28,867] - WARN - login api 失败，username='alice' 密码错误
[2026-08-18 11:25:28,867] - INFO - 调用 login api, username='bob'
[2026-08-18 11:25:29,236] - INFO - 调用 login api, username='alice'
[2026-08-18 11:25:29,712] - INFO - login api success, username='bob' 登陆成功
[2026-08-18 11:25:29,771] - WARN - login api 失败，username='alice' 密码错误
[2026-08-18 11:25:30,028] - INFO - 调用 login api, username='alice'
[2026-08-18 11:25:30,169] - WARN - username='bob' 支付失败，余额不足
[2026-08-18 11:25:30,334] - INFO - login api success, username='alice' 登陆成功
[2026-08-18 11:25:30,801] - INFO - username='alice' 支付成功
```

这里有两个用户，并发访问了登录和支付接口，传统的日志至少包含几个部分：时间戳，日志级别和日志信息。

### 问题一：并发量
一旦访问的并发数激增，那么所有日志信息就像是意大利面一样纠缠不清，为了排查用户反馈的支付异常的问题，只能求助于 grep、awk 等工具。

### 问题二：上下文丢失

上面的示例中显示了 bob 支付失败，并且打出了原因，但是这个原因依然非常模糊，bob 使用的什么方式支付的？为了购买什么产品支付的？产品在那个时间下，价格如何？

### 问题三：非结构化

当碰到第二个问题后，我们可以选择在 message 中记录下更多的信息：

```
[2026-08-18 11:25:30,169] - WARN - username='bob' 支付失败，失败原因：余额不足，支付方式：支付宝，购买产品：XX品牌鼠标，产品价格：128
```

然后后续再通过 grep 的时候能看到更多的上下文信息，但由于把所有上下文信息都塞在了一个 message 中，导致 message 包含的信息过多，而且是纯文本的形式，没有任何约束性的格式，导致我们依然无法回答更加深入的问题：今天有多少用户是支付失败并且支付方式是支付宝的？为了回答这个简单的问题，需要在一个巨大的日志文本中编写复杂的 shell 命令：

```shell
grep '支付失败' app.log | grep '支付方式：支付宝' | sed -n "s/.*username='\([^']*\)'.*/\1/p"

or

awk '/支付失败/ && /支付方式：支付宝/ {
    match($0, /username='\''[^'\'']+'\''/)
    print substr($0, RSTART + 10, RLENGTH - 11)
}' app.log
```

这些脚本虽然能够实现功能，但是编写和阅读都非常困难，根本原因就是没有约束的非结构化文本，写起来很简单，但将所有的复杂度都丢给了后续的处理和排查，这在大型工程项目里，是不明智的。

我们应该把日志复杂度的处理都尽可能放在程序编写和设计阶段，在后期处理的时候，就会有更加丰富的工具选择。

---

## 日志结构化

OpenTelemetry 对于结构化、半结构化以及非结构化有着明确的[例子](https://opentelemetry.io/docs/concepts/signals/logs/#structured-unstructured-and-semistructured-logs)。

### 非结构化

非结构化就是传统的日志（Common Log Format, CLF），也就是上一节提到的案例。

大规模的非结构化的日志虽然局部上看，人类友好可读，但下游系统难以解析，通常要借助正则或者其他分析文本的工具，要根据一些字段分组和排序也会非常低效。

### 结构化日志

结构化日志是一种具有定义一致模式或类型字段的日志,下游系统能够可靠地解析和解释。文本编码可以是 JSON、protobub 或其他格式,但使日志结构化的是存在一个稳定的模式(字段名称、类型和语义),而不仅仅是它是有效的 JSON。例如,结构化的 JSON 日志可能如下所示：

```json
{
  "timestamp": "2024-08-04T12:34:56.789Z",
  "level": "INFO",
  "service": "user-authentication",
  "environment": "production",
  "message": "User login successful",
  "context": {
    "userId": "12345",
    "username": "johndoe",
    "ipAddress": "192.168.1.1",
    "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36"
  },
  "transactionId": "abcd-efgh-ijkl-mnop",
  "duration": 200,
  "request": {
    "method": "POST",
    "url": "/api/v1/login",
    "headers": {
      "Content-Type": "application/json",
      "Accept": "application/json"
    },
    "body": {
      "username": "johndoe",
      "password": "******"
    }
  },
  "response": {
    "statusCode": 200,
    "body": {
      "success": true,
      "token": "jwt-token-here"
    }
  }
}
```

结构化日志不仅仅是经过 json 格式化的日志，还必须要求所有字段的模式和语义必须固定统一，下游系统才能可靠的分析日志。

### 半结构化日志

半结构化日志通常是键值对或者其他机器友好的可解析类型，但相比结构化日志，它的模式和语义都不一定是固定的，往往在流向分析系统前，还需要进行一些处理和映射，下面就是一个例子：

```
2024-08-04T12:45:23Z level=ERROR service=user-authentication userId=12345 action=login message="Failed login attempt" error="Invalid password" ipAddress=192.168.1.1 userAgent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36"
```

---

## 字段

Log Record 就是一次事件的记录，不是简单的一行文本，而是结构化的对象，包含两种类型的字段：

1. 拥有固定类型和具体含义的顶层字段，可以理解为必填字段
2. 可扩展的属性字段，每个具体业务场景的选填字段

顶层字段必须要在所有记录中具有相同的语义，也是所有日志记录必填字段，例如：时间戳、日志级别等。

OTel 的顶层字段包含：

- Timestamp：时间戳，事件何时发生
- ObservedTimestamp：时间戳，事件何时被观测到
- TraceId：链路追踪的 id
- SpanId：同一链路下，某个具体的步骤的 id, 该字段属于 TraceId 的附属属性，可选，如果 TraceId 不存在或者无效，该字段应被忽略
- TraceFlags：一个标记位，用于采样
- SeverityText：日志级别（文本）
- SeverityNumber：日志级别（数字）较小的数值对应较不严重的事件(例如调试事件),较大的数值对应更严重的事件(如错误和关键事件)
- Body：包含日志记录正文的值。例如,可以是一个可读的字符串消息(包括多行),以自由形式描述事件,也可以是由数组和其他值映射组成的结构化数据。
- Resource：描述日志的来源,即资源。来自同一事件源的多个事件可能随时间而发生,且其都具有相同的资源价值。例如,可以包含有关发布记录的应用程序或应用程序运行基础设施的信息。
- InstrumentationScope：描述发出这条日志的 instrumentation 范围（通常是库名 + 版本）。
- Attributes：有关具体事件事件的更多信息。与针对特定来源固定的 Resource 不同,来自同一来源的事件每次出现的属性可能有所不同。可以包含有关请求上下文的信息。
- EventName：标识事件类/类型的名称。此名称应唯一地标识事件结构(包括属性和身体)。具有非空事件名称的日志记录是事件。

精确定义多个顶层字段，可以将过去我们一切都塞在一个 message 文本的坏习惯抽离出来，让 message 变得更轻，把一些统一的、几乎所有系统都一致的字段抽离成顶层字段。

Python logging 库中日志数据抽象是 LogRecord，其字段在[文档](https://docs.python.org/3/library/logging.html#logrecord-attributes)中可以找到。

综合 OTel 和 Python 的 LogRecord 模型来看，我认为在普通的系统之中至少有以下一些字段：

1. timestamp
2. level
3. resource
4. message
5. event\_name
6. exception
7. trace\_id
8. span\_id
9. attributes

---

## dictConfig

为了实现 json 格式化，可以自己手动实现一个formatter,也可引入第三方的依赖——(python-json-logger)[https://pypi.org/project/python-json-logger/]。



---

## 参考

1. https://opentelemetry.io/docs/specs/otel/logs/data-model/
2. https://loggingsucks.com/
3. https://opentelemetry.io/docs/concepts/signals/logs/
4. https://opentelemetry.io/docs/specs/semconv/http/http-spans/
5. https://python-observability.com/
