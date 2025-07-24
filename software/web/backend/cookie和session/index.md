---
title: "cookie和session"
date: 2025-07-23T16:21:00+08:00
summary: "服务器辨认客户端身份的基础"
---

## 前言

HTTP 协议是无状态的，所谓无状态，指的是服务器无法确认多个请求之间是否存在管理，或者多个请求是否由同一个用户发出。这种简单的设计在静态类型页面（没有太多交互）的网站上工作良好，但是对于动态网站（根据不同的用户生成不同的页面）来说，无状态就很棘手了。

一些动态网站（比如：电商网站，博客平台等）需要标志出一个用户在访问一项业务（也许是添加多个商品到购物车，也许是浏览了几个页面需要记录历史浏览状态）的时候，需要借助 HTTP 的 cookie 和 session 技术来完成。

---

## cookie

cookie 是一种在 web 服务器和 web 浏览器之间传递信息的技术，最早由美国的李维·蒙塔利（Lou Montulli）于 1994 年发明，通过在 HTTP response headers 中添加一个 set-cookie 字段来完成 web 服务器在 web 浏览器上保存一些信息，等下一次 web 浏览器发送请求时，就会携带 set-cookie 里的内容（放在 request header 的 cookie 字段）交给 web 服务器。

可以先把 cookie 简单的想象一个浏览器的存储功能，存储了针对于不同的网站的键值对，在 chrome/firefox/edge 等浏览器的 settings-隐私与安全中，可以找到所有网站的 cookies：

![](./images/1.jpg)

首先 web 浏览器可以通过这个字段，保存一些关于当前使用该浏览器的用户的个人信息，比如：用户设置偏好，用户名，购物车列表等等，这样用户打开特定网站时，web 浏览器就可以通过这些 cookie 内容，展示特定于当前用户的页面。

另外一方面，web 服务器为了从成千上万的 HTTP 请求中，辨认出哪几个 HTTP 请求属于某一个特定的用户或者浏览器，就可以在 cookie 里面塞一个 id（比如一个 UUID），这种模式就引申出了 cookie 的另外一个用法——维护 session。

---

## session

cookie 本身并不是为了解决“在浏览器上存东西”而被发明，它的出现是为了解决 HTTP 协议无状态特性的问题。在网络的早期，当没有其他选择时，cookie 被用于数据存储目的（也就是上一小节提到的用法），现在建议使用新式存储 API，例如 Web Storage API (localStorage，sessionStorage) 或者 IndexedDB。

如果说 cookie 是 HTTP 中一个实际的 header 字段，那么 session 更像是一个过程，它基于 cookie 实现：

1. 用户使用账号名密码登录
2. 登陆成功后，服务器为该用户生成一个 SessionId，并且在服务器内存中存储了这个 SessionId 对应的用户信息。
3. 服务器通过 set-cookie 把 SessionId 告诉浏览器，浏览器存储到 cookie 中。
4. 浏览器之后每一次访问该站点，会在 cookie 中携带这个 SessionId 向服务器表明自己的身份。

---

## 安全性

---

## 参考

1. https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies
2. https://http.dev/cookies
3. https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview#http_is_stateless_but_not_sessionless
4. https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Set-Cookie
5. https://zhuanlan.zhihu.com/p/688144675