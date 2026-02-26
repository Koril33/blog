---
title: "关于前后端接口规范的讨论"
date: 2026-02-26T16:29:54
summary: "混乱产生于没有标准时的各行其道"
---

## 前言

在看到左耳朵耗子的这篇文章[我做系统架构的一些原则](https://coolshell.cn/articles/21672.html)的关于 HTTP Response Status Code 的讨论，颇有感触：

> 比如：我经常性的见到很多公司的系统既没有服从业界标准，也没有形成自己公司的标准，感觉就像一群乌合之众一样。最典型的例子就是 HTTP 调用的状态返回码。业内给你的标准是 200表示成功，3xx 跳转，4xx 表示调用端出错，5xx 表示服务端出错，我实在是不明白为什么无论成功和失败大家都喜欢返回 200，然后在 body 里指出是否error（前两年我在微信公众号里看到一个有一定名气的互联网老兵推荐使用无论正确还是出错都返回 200 的做法，我在后台再三确认后，我发现这样的架构师真是害人不浅）。这样做最大的问题是——监控系统将在一种低效的状态下工作。监控系统需要把所有的网络请求包打开后才知道是否是错误，而且完全不知道是调用端出错还是服务端出错，于是一些像重试或熔断这样的控制系统完全不知道怎么搞（如果是 4xx错，那么重试或熔断是没有意义的，只有 5xx 才有意义）。有时候，我会有种越活越退步的感觉，错误码设计这种最基本最基础的东西为什么会没有？并且一个公司会任由着大家乱来？这些基础技能怎么就这样丢掉了？

其实只要是做 web 开发的，无论前端还是后端，只要换的公司数量够多，或者看过的开源框架数量够多，都会碰到这个问题，因为对于 HTTP Code 该不该用，该怎么用，从来都没有一个统一的标准，有的团队坚持 RESTful，有的团队坚持只有 GET+POST，足够宽松的 HTTP 协议并没有对前后端通信中产生的业务问题做出什么约束，所以大家各行其道。

后来慢慢才有一种体会，这种没有标准的标准是需要团队自己来确定的，就像代码规范一样（命名规范，大小写规范，缩进）没有一个绝对正确的原则，只有团队统一一个标准，并且团队后续的每一位成员参与的每一项目都按照该标准来执行，才是最佳的实践。

---

## 思考

网上争论的帖子很多，可以去 V2EX 搜索下，主要是因为哪种方案都能用，大家场景不同，公司规模不同，使用的框架和基础设施不同，没有说 A 方案一定可以，而 B 方案就绝对不行的。

架构师的一个主要职责就是在这里，理清团队中模糊的规范，并且保证这个规范在一定时间内是适用于大部分自家公司的场景的，这种基础工作，在大规模的团队中，是不能等到前端、后端、运维等多方因为“Code 到底在 HTTP Status Code 还是在 Response Body JSON Code 中取来进行业务判断”的问题而争吵时才去定下规则。

---

## 方案

这里写下的是我个人，在自己的项目中使用的规范，并不一定适用于其他项目，仅供参考。

### 成功时返回 JSON

前端仅仅需要的数据，所以可以摈弃过去的这种冗余结构：

```json
{
	"success": true,
	"code": 1001,
	"data": {
		"username": "alice",
		"age": 11
	}
}
```

直接返回数据：

```json
{
	"username": "alice",
	"age": 11
}
```

因为在调用类似分页搜索接口时，success 为 true，一般都会有数据，前端直接拿着 Response JSON 解析数据就行了，不需要再做 response_body.json().data 这种操作，code 在成功拿到数据的情况下，也没有存在的必要。

### 失败时的 JSON

这才是大家争论的重头戏，因为现实中，工程师写完业务逻辑仅仅只是完成了 20% 的工作量，剩下的 80% 的工作量都是在和各种奇奇怪怪的报错和异常以及意料之外的情况做搏斗。

从 HTTP 官方的语义来看，4XX 表示客户端异常，5XX 表示服务端异常，这在一些小项目中是够用的，但是到了一些前端有更加丰富的显示的项目里，就不够用了。

所以对于失败的情况，HTTP Status Code 可以遵守官方的约定，在 JSON 中参考[RFC 7807](https://www.rfc-editor.org/rfc/rfc7807)

下面举几个失败的返回例子。

客户端错误

字段校验失败：

```
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/validation-error",
  "title": "请求参数验证失败",
  "status": 400,
  "detail": "请求体中的 'email' 字段格式不正确",
  "instance": "/logs/2024/abc123",
  "field": "email",
  "rejected_value": "invalid-email"
}

```

批量校验字段失败：

```
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/validation-error",
  "title": "请求参数验证失败",
  "status": 400,
  "invalid_params": [
    {
      "name": "age",
      "reason": "必须是正整数"
    },
    {
      "name": "color",
      "reason": "必须是 'green', 'red' 或 'blue'"
    }
  ]
}
```

余额不足：

```
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/insufficient-credit",
  "title": "账户余额不足",
  "status": 403,
  "detail": "当前余额 30 元，但此次操作需要 50 元",
  "instance": "/account/12345/transactions/tx-789",
  "balance": 30,
  "required": 50,
  "accounts": ["/account/12345", "/account/67890"]
}
```

服务端错误

数据库连接超时：

```
HTTP/1.1 500 Internal Server Error
Content-Type: application/problem+json

{
  "type": "about:blank",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "数据库连接超时，请稍后重试",
  "instance": "/errors/2024-01-15/uuid-xyz"
}
```

这些字段只是 RFC 标准里的参考，可以自定义失败返回 JSON，在业务失败的场景下，JSON 中的 Code 才有具体意义，以用户登录失败来说，可能是用户名密码错误，可能是用户被锁定，也可能是触发登陆风控，HTTP Code 对于这些失败场景都返回同一个 401，那么前端就没法做更加细粒度的展示了，所以在失败响应 JSON 中的 Code 就是给前端某一业务错误下的更加具体的标记。

这种 error code 的定义可以参考各个大公司的规范，比如新浪：https://open.weibo.com/wiki/Error_code

---

## 参考

1. https://v2ex.com/t/725240
2. https://v2ex.com/t/791754
3. https://v2ex.com/t/1018946
4. https://v2ex.com/t/824337
5. https://v2ex.com/t/676678
6. https://v2ex.com/t/899875