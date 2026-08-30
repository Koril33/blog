---
title: "请求外部接口需要做的准备"
date: 2026-08-28T21:06:03
summary: "外部的接口是未知的，不确定的，不稳定的，我们需要做足准备"
---

## 前言

系统之间难免会产生依赖，如果依赖的是比较流行的开源库，那么在曝出特大漏洞前，我们多可以安心使用，但如果依赖的是第三方接口，那么就需要小心，首当其冲的就是各种诡异的网络问题，接下来还需要面对认证问题，限流问题，处理超时问题，以及第三方接口随时可能会改变数据模型的风险。

调用外部接口（本文指的是基于 HTTP 协议的 Web API，当然思想是抽象的，也可以扩展到其他协议类型），本质上是在和一个不可控的远程系统打交道，在面对未知系统的复杂度问题前，我们先要仔细审视手头的工具，看看能做好哪些防御式编程的工作。

---

## 第三方接口

为什么外部接口（或者叫第三方接口）是不可控的未知，是因为代码往往非开源，我们无法审查源码库，我们仅能从一个 API 接口获取所有“狭窄”的信息。

外部接口的未知和不可控体现在以下一些方面：

- 网络不可控，随时可能超时、中断。
- 对方服务状态不可控，随时可能 Server Internal Error（status code == 500）。
- 性能不可控，对方服务的响应速度可能时快时慢。
- 数据格式不可控，对方可能随时修改返回的数据模型。
- 服务的限流、升级维护、认证机制变化都无法掌控。

调用第三方 API，本质上是在一个不确定的系统边界上进行通信。我们在完成业务逻辑代码的编写后，第二个要解决的问题就是：当 API 不按照预期工作时，我们自己的系统怎么办。

---

## 可能的错误

### 网络

网络异常，可能是己方网络故障，也可能是对方网络故障，从请求发起到受到响应信息，中间经历了很多过程：

```

Python or Java or Go or Other Programming Language
  ↓
HTTP Client 库
  ↓
操作系统
  ↓
网卡
  ↓
路由器
  ↓
DNS
  ↓
ISP
  ↓
Internet
  ↓
第三方 CDN / LB
  ↓
第三方服务器

```

可以参考：https://200ms.thenodebook.com/，这个网站可视化的描绘了按下屏幕中一个下单按钮到显示器显示订单成功之间 200ms 的网络历程。

Python requests 库中对于网络异常有了一个简单的分类：

```
Exception
└── IOError
    └── RequestException
        ├── InvalidJSONError
        │   └── JSONDecodeError
        │
        ├── HTTPError
        │
        ├── ConnectionError
        │   ├── ProxyError
        │   ├── SSLError
        │   └── ConnectTimeout
        │       └── Timeout
        │
        ├── Timeout
        │   └── ReadTimeout
        │
        ├── URLRequired
        ├── TooManyRedirects
        ├── MissingSchema
        ├── InvalidSchema
        ├── InvalidURL
        │   └── InvalidProxyURL
        ├── InvalidHeader
        ├── ChunkedEncodingError
        ├── ContentDecodingError
        ├── StreamConsumedError
        ├── RetryError
        └── UnrewindableBodyError

Warning
└── RequestsWarning
    ├── FileModeWarning
    └── RequestsDependencyWarning
```

里面包含一些客户端的异常，这些异常是需要在客户端（也就是我们自己调用方这边）极力避免的：

- MissingSchema：没有指定协议（http、https），例如：requests.get("")
- InvalidSchema：指定了一个无法识别的协议，例如：requests.get("dummy://")
- InvalidURL：URL 无效，例如：requests.get("http://")
- InvalidProxyURL：代理的 URL 无效，例如：requests.get("http://123.com", proxies={"http": "http://"})
- InvalidHeader：headers无效，例如：requests.get("http://127.0.0.1", headers={"dummy": "ok\nboom"})

这几个 Missing 和 Invalid 可以在客户端避免发生，是我们在编写代码的层面上，完全可以控制的。

其他的异常大多和服务端有关（但是不绝对，己方网络有问题，也会导致下面的一些异常发生）

首先是连接异常和连接超时，连接异常是 ConnectionError，超时异常是 Timeout，超时可以分为 ConnectTimeout 和 ReadTimeout。

ConnectionError 的原因可能有很多，我们可以访问一个压根不存在的网址来模拟：

```python

>>> requests.get("http://fjdslafjklsda.dummy", proxies={"http": ""})
Traceback (most recent call last):
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connection.py", line 204, in _new_conn
    sock = connection.create_connection(
        (self._dns_host, self.port),
    ...<2 lines>...
        socket_options=self.socket_options,
    )
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/util/connection.py", line 60, in create_connection
    for res in socket.getaddrinfo(host, port, family, socket.SOCK_STREAM):
               ~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/socket.py", line 983, in getaddrinfo
    for res in _socket.getaddrinfo(host, port, family, type, proto, flags):
               ~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
socket.gaierror: [Errno -2] Name or service not known

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connectionpool.py", line 788, in urlopen
    response = self._make_request(
        conn,
    ...<10 lines>...
        **response_kw,
    )
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connectionpool.py", line 493, in _make_request
    conn.request(
    ~~~~~~~~~~~~^
        method,
        ^^^^^^^
    ...<6 lines>...
        enforce_content_length=enforce_content_length,
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    )
    ^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connection.py", line 500, in request
    self.endheaders()
    ~~~~~~~~~~~~~~~^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/http/client.py", line 1333, in endheaders
    self._send_output(message_body, encode_chunked=encode_chunked)
    ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/http/client.py", line 1093, in _send_output
    self.send(msg)
    ~~~~~~~~~^^^^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/http/client.py", line 1037, in send
    self.connect()
    ~~~~~~~~~~~~^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connection.py", line 331, in connect
    self.sock = self._new_conn()
                ~~~~~~~~~~~~~~^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connection.py", line 211, in _new_conn
    raise NameResolutionError(self.host, self, e) from e
urllib3.exceptions.NameResolutionError: HTTPConnection(host='fjdslafjklsda.dummy', port=80): Failed to resolve 'fjdslafjklsda.dummy' ([Errno -2] Name or service not known)

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/adapters.py", line 696, in send
    resp = conn.urlopen(
        method=request.method,
    ...<9 lines>...
        chunked=chunked,
    )
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connectionpool.py", line 842, in urlopen
    retries = retries.increment(
        method, url, error=new_e, _pool=self, _stacktrace=sys.exc_info()[2]
    )
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/util/retry.py", line 543, in increment
    raise MaxRetryError(_pool, url, reason) from reason  # type: ignore[arg-type]
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
urllib3.exceptions.MaxRetryError: HTTPConnectionPool(host='fjdslafjklsda.dummy', port=80): Max retries exceeded with url: / (Caused by NameResolutionError("HTTPConnection(host='fjdslafjklsda.dummy', port=80): Failed to resolve 'fjdslafjklsda.dummy' ([Errno -2] Name or service not known)"))

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<python-input-40>", line 1, in <module>
    requests.get("http://fjdslafjklsda.dummy", proxies={"http": ""})
    ~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/api.py", line 87, in get
    return request("get", url, params=params, **kwargs)
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/api.py", line 71, in request
    return session.request(method=method, url=url, **kwargs)
           ~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/sessions.py", line 651, in request
    resp = self.send(prep, **send_kwargs)
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/sessions.py", line 784, in send
    r = adapter.send(request, **kwargs)
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/adapters.py", line 729, in send
    raise ConnectionError(e, request=request)
requests.exceptions.ConnectionError: HTTPConnectionPool(host='fjdslafjklsda.dummy', port=80): Max retries exceeded with url: / (Caused by NameResolutionError("HTTPConnection(host='fjdslafjklsda.dummy', port=80): Failed to resolve 'fjdslafjklsda.dummy' ([Errno -2] Name or service not known)"))

```

这里产生了一个异常链，最上层（也就是靠近我们的这一层）是 requests 的 ConnectionError，因为 requests 库实际上是 urllib3 库的高级封装，所以内部异常是 urllib3 发出的，再底层就是 socket 的异常，也就是无法成功进行这个网址的 DNS 解析：

```
socket
  │
  └── socket.gaierror
          │
          ▼
urllib3
  │
  └── NameResolutionError
          │
          ▼
urllib3
  │
  └── MaxRetryError
          │
          ▼
requests
  │
  └── ConnectionError
```

这是由 DNS 异常引发的 ConnctionError, 我们还可以通过访问一个不存在的端口服务（DNS不会异常，但是指定的服务端口实际上没有监听着的服务）：

```python

>>> requests.get("http://127.0.0.1:8765", proxies={"http": ""})
Traceback (most recent call last):
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connection.py", line 204, in _new_conn
    sock = connection.create_connection(
        (self._dns_host, self.port),
    ...<2 lines>...
        socket_options=self.socket_options,
    )
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/util/connection.py", line 85, in create_connection
    raise err
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/util/connection.py", line 73, in create_connection
    sock.connect(sa)
    ~~~~~~~~~~~~^^^^
ConnectionRefusedError: [Errno 111] Connection refused

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connectionpool.py", line 788, in urlopen
    response = self._make_request(
        conn,
    ...<10 lines>...
        **response_kw,
    )
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connectionpool.py", line 493, in _make_request
    conn.request(
    ~~~~~~~~~~~~^
        method,
        ^^^^^^^
    ...<6 lines>...
        enforce_content_length=enforce_content_length,
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    )
    ^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connection.py", line 500, in request
    self.endheaders()
    ~~~~~~~~~~~~~~~^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/http/client.py", line 1333, in endheaders
    self._send_output(message_body, encode_chunked=encode_chunked)
    ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/http/client.py", line 1093, in _send_output
    self.send(msg)
    ~~~~~~~~~^^^^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/http/client.py", line 1037, in send
    self.connect()
    ~~~~~~~~~~~~^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connection.py", line 331, in connect
    self.sock = self._new_conn()
                ~~~~~~~~~~~~~~^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connection.py", line 219, in _new_conn
    raise NewConnectionError(
        self, f"Failed to establish a new connection: {e}"
    ) from e
urllib3.exceptions.NewConnectionError: HTTPConnection(host='127.0.0.1', port=8765): Failed to establish a new connection: [Errno 111] Connection refused

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/adapters.py", line 696, in send
    resp = conn.urlopen(
        method=request.method,
    ...<9 lines>...
        chunked=chunked,
    )
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/connectionpool.py", line 842, in urlopen
    retries = retries.increment(
        method, url, error=new_e, _pool=self, _stacktrace=sys.exc_info()[2]
    )
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/urllib3/util/retry.py", line 543, in increment
    raise MaxRetryError(_pool, url, reason) from reason  # type: ignore[arg-type]
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
urllib3.exceptions.MaxRetryError: HTTPConnectionPool(host='127.0.0.1', port=8765): Max retries exceeded with url: / (Caused by NewConnectionError("HTTPConnection(host='127.0.0.1', port=8765): Failed to establish a new connection: [Errno 111] Connection refused"))

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<python-input-41>", line 1, in <module>
    requests.get("http://127.0.0.1:8765", proxies={"http": ""})
    ~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/api.py", line 87, in get
    return request("get", url, params=params, **kwargs)
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/api.py", line 71, in request
    return session.request(method=method, url=url, **kwargs)
           ~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/sessions.py", line 651, in request
    resp = self.send(prep, **send_kwargs)
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/sessions.py", line 784, in send
    r = adapter.send(request, **kwargs)
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/adapters.py", line 729, in send
    raise ConnectionError(e, request=request)
requests.exceptions.ConnectionError: HTTPConnectionPool(host='127.0.0.1', port=8765): Max retries exceeded with url: / (Caused by NewConnectionError("HTTPConnection(host='127.0.0.1', port=8765): Failed to establish a new connection: [Errno 111] Connection refused"))

```

本地的 8765 实际上没有运行的服务，所以异常链是：

```
socket.connect()
      │
      │  TCP 连接被拒绝
      ▼
ConnectionRefusedError
[Errno 111] Connection refused
      │
      │  raise NewConnectionError(...) from e
      ▼
urllib3.exceptions.NewConnectionError
Failed to establish a new connection
      │
      │  Retry.increment()
      │  raise MaxRetryError(...) from reason
      ▼
urllib3.exceptions.MaxRetryError
Max retries exceeded
      │
      │  requests.adapters.HTTPAdapter.send()
      │  raise ConnectionError(e, request=request)
      ▼
requests.exceptions.ConnectionError
      │
      ▼
requests.get()
```

在实际工作中，TCP 连接被拒绝（Connection refused）通常比 DNS 解析异常更容易遇到。例如目标服务重启、服务进程异常退出，或者目标端口没有服务监听时，客户端建立 TCP 连接就可能收到 Connection refused。

不过，第三方服务停机或维护并不一定表现为 Connection refused。如果前面的负载均衡、反向代理或 API Gateway 仍然正常工作，也可能返回 HTTP 503、502 等状态码；如果网络层直接丢弃数据包，则可能表现为连接超时。

ConnectTimeout 是在指定的时间内，没有成功建立 TCP 连接抛出的异常。

ReadTimeout 和 ConnectTimeout 不同，ReadTimeout 是在 TCP 连接已经建立的情况下，由于对方服务响应太慢导致的异常。

requests 中有个请求参数是 timeout，在实际生产环境代码**必须**要指定该参数，官方文档的原文是：

> You can tell Requests to stop waiting for a response after a given number of seconds with the timeout parameter. Nearly all production code should use this parameter in nearly all requests. Failure to do so can cause your program to hang indefinitely.

如果生产环境不指定 timeout，那么线程就可能永远阻塞等待在这个 IO 调用里，相当于把控制权交给了第三方服务，这是非常危险的，如果请求第三方服务的次数比较频繁，大量线程可能就会全部阻塞，系统资源被耗尽。

timeout 可以接受单个 float 类型的数字，也可以传递包含两个元素的 tuple。

timeout 会指定两个超时时间，一个是连接超时时间（connection timeout seconds），另一个是接受（读取）服务端数据超时时间（read timeout seconds）。

```python
# 传递单个 float 类型表示连接超时时间和读取超时时间都是同一个值，下面的 timeout=3 等价于 timeout=(3, 3)
requests.get("http://localhost:5000/api/data", timeout=3)

# 传递两个元素的 tuple 表示分别指定连接超时时间和读取超时时间
requests.get("http://localhost:5000/api/data", timeout=(3, 5))
```

连接超时时间就是之前提到的 TCP 建立连接的时间，超过这个时间无法建立 TCP 连接就会爆出 ConnectTimeout 异常。

读取超时时间，则是服务器连续给定的时间内，没有返回任何数据就会爆出 ReadTimeout 异常，需要注意的是如果只是服务端持续在传输数据，但是传输的较慢或者数据很大，传输时间很久，是不会触发 ReadTimeout 异常的，举个例子：如果服务器每 15 秒传输 1 个字节，即使请求需要几小时才能完成，指定了的 20 秒的读取超时也不会触发。

ReadTimeout 异常在连接建立后，连续两次读取之间没有任何数据到达超过 n 秒时触发，典型场景：

- 服务器一直没返回任何字节（最常见是等不到响应头/首个字节）。
- 服务器先发了一部分，然后中途卡住超过 n 秒。
- 网络抖动导致某一段长时间没有数据。

所以这里要认识到 timeout 指定的这两个超时时间，和总的请求时间（请求发起开始到接收到最后一个字节断开连接结束）是没有什么关联关系的。

如果想硬性限制整个请求的总时长，requests 的 timeout 做不到，得在应用代码层面通过 time.monotonic() 计时，主动在流读取过程中处理超时终止。

### HTTP

HTTP 层面的异常多体现在 response status code,也就是响应的状态码，2xx 一般是成功，4xx 是客户端异常，比如：400 传参有问题，401 未登录，403 权限不足，404 页面未找到，429 访问过于频繁被限流，5xx 指的是服务端异常。

requests 受到响应后，如果响应的 status code 不是 2xx, 并不会抛出 HTTPError 的异常，如果想要在非 2xx 的情况下抛出 HTTPError,需要调用 raise\_for\_status() 的方法。

例如：

```python

>>> requests.get("https://example.com/api/data")
<Response [401]>
>>> response = requests.get("https://example.com/api/data")
>>> response.json()
{'message': 'Unauthorized'}
>>> response.raise_for_status()
Traceback (most recent call last):
  File "<python-input-63>", line 1, in <module>
    response.raise_for_status()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/models.py", line 1167, in raise_for_status
    raise HTTPError(http_error_msg, response=self)
requests.exceptions.HTTPError: 401 Client Error: Unauthorized for url: https://example.com/api/data

```

### 业务层

在一些不那么严谨的接口里，HTTP 异常状态码和业务异常状态码没有统一的使用规范，经常混在一起使用，比如：

```

HTTP status code = 200 OK

response body:

{
  "success": false,
  "code": 112233,
  "message": "该账号不存在"
  "data": null
}


```

虽然 HTTP status code 是 200，但是接口并没有成功，因为异常的状态放在了响应体中，而不是让 HTTP status code = 400。所以在解析到 200 的响应后也不能掉以轻心，真正的异常可能放在了 body 中。

### 数据模型

第三方服务的数据模型也是不能轻易相信的，抱着接口数据模型随时会修改的预期，碰到 KeyError、NPE 的时候就不会那么惊讶了。

跟 requests 相关的异常包括 InvalidJSONError 和 JSONDecodeError，后者是前者的子类型。

比如第三方服务向来都是 json 格式的 body，有一天突然换成了返回字符串的形式：

```python
>>> response = requests.get("http://localhost:5000/plain", timeout=(3, 5))
>>> response.text
'ok'
>>> response.json()
Traceback (most recent call last):
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/models.py", line 1116, in json
    return complexjson.loads(self.text, **kwargs)
           ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/json/__init__.py", line 346, in loads
    return _default_decoder.decode(s)
           ~~~~~~~~~~~~~~~~~~~~~~~^^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/json/decoder.py", line 345, in decode
    obj, end = self.raw_decode(s, idx=_w(s, 0).end())
               ~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/koril/.local/share/uv/python/cpython-3.14.0-linux-x86_64-gnu/lib/python3.14/json/decoder.py", line 363, in raw_decode
    raise JSONDecodeError("Expecting value", s, err.value) from None
json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<python-input-57>", line 1, in <module>
    response.json()
    ~~~~~~~~~~~~~^^
  File "/home/koril/project/study/study-requests/.venv/lib/python3.14/site-packages/requests/models.py", line 1120, in json
    raise RequestsJSONDecodeError(e.msg, e.doc, e.pos)
requests.exceptions.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
>>> 
```

---

## 手段

### 控制时间

我们不能永远阻塞在外部服务的网络调用上。任何请求必须要指定 timeout 超时时间。

### 有原则的重试

重试是一种有效的手段，但需要有一定的策略。有些异常可以重试，有些异常没必要重试。

碰到网络波动相关的错误，比如 ReadTimeout, 或者服务器限流 429, 服务器短暂的不可以 500, 这些情况下，客户端可以进行“礼貌”的重试，这里的“礼貌”指的是有限次数，以及每次重试要间隔一定的时间。

当一个请求/任务失败时，不要让客户端立刻、连续、高频地再次请求，避免把一个暂时故障放大成更严重的故障，高频连续的重试会让服务端不堪重负，可以采用指数退避的策略。

有些异常没必要重试，比如已经从网关接口获取到了服务器正在维修之类的信息。

更多的异常是需要客户端修改一些参数后，再尝试重试，这种重试才是有实践意义的，比如碰到 400 考虑修改传参后再重试，碰到 401 和 403 考虑是认证授权的问题，需要更换账号或者获取对应权限后再重试。

所以重试是有前提条件的，大致分为三类：

- 网络异常、限流、服务短暂不可用：遵循指数退避的原则，礼貌的重试
- 客户端传参、权限异常：尝试修改参数后再重试
- 确定性失败，服务端明确不可用：没必要重试，记录日志，或者发起告警即可

### 接受错误

不要让外部服务的问题，拖垮自己的系统。

以外部服务会改变数据模型来说，传统的 dict\["key"\] 很容易导致 KeyError,过于依赖稳定的 JSON 模型还会导致 NPE（NoneType 的问题）。

我们需要以下一些工具来处理常常变更的模型：

纯安全取值           │ toolz / cytoolz
点号路径小工具       │ dotted-notation、nested-property
声明式访问+容错+转换 │ glom
路径操作+通配符      │ dpath
查询语言             │ jmespath
查询语言（最强）     │ jsonpath-ng

### 接受不确定

读取数据超时（ReadTimeout）不一定是错误或者失败，更可能是一种不确定的状态。

一个 POST 提交订单，在后台可能已经记录落库了，业务逻辑都执行完毕，只是最后没有成功传输数据返回给客户端。

### 保留证据

接口调用失败时，排查问题往往依赖请求和响应的上下文，因此需要在请求过程中保留足够的诊断信息。不过，“保留证据”并不意味着把完整的 URL、Headers、请求参数和响应内容无差别地写入日志。

首先，接口返回 200 也不代表请求一定符合业务预期。例如参数虽然合法，但值填写错误，服务端仍然可能正常返回。因此，即使请求成功，也应该保留一些能够用于追踪和审计的信息，例如请求目标、HTTP 方法、状态码、耗时、业务标识、trace_id、参数摘要等。

但对于成功请求，一般没有必要记录完整的请求和响应内容。完整记录不仅会显著增加日志量，还可能泄漏敏感信息，例如：

- Authorization、Cookie、API Key、Session
- URL Query 中的 Token、签名等认证参数
- 用户密码、手机号、邮箱、身份证等敏感字段
- 请求或响应中的大段业务数据。

因此，更合理的做法是对日志进行分级处理。

对于正常请求，只记录必要的摘要信息，例如：

method=GET
host=api.example.com
path=/v1/schedules
status_code=200
duration_ms=326
trace_id=...
request_id=...

必要时可以记录部分非敏感业务参数，但不应默认记录完整请求体和响应体。

对于异常请求，则可以增加诊断信息，例如请求参数、响应状态码、响应内容以及异常堆栈，但仍然需要经过脱敏、过滤和长度限制。例如：

Authorization: Bearer ***
Cookie: ***
password: ***
access_token: ***

同时，应限制请求体和响应体的最大日志长度。第三方接口可能返回几百 KB 甚至数 MB 的数据，如果在异常时完整写入日志，不但会造成日志膨胀，还可能进一步影响故障期间的系统性能。例如可以只保留前几 KB，并明确标记内容已经被截断。

对于特别重要但又不适合长期写入普通日志的原始请求和响应，可以采用单独的诊断存储机制，例如设置较短的保留时间、严格的访问权限，并通过 trace_id 或 request_id 与普通日志关联，而不是直接把所有内容永久记录下来。

因此，“保留证据”应该遵循几个基本原则：

1. 正常请求记录摘要，异常请求记录更多上下文
2. 敏感信息默认不记录，必须记录时先脱敏
3. 请求体和响应体必须限制大小，避免日志失控
4. 使用 trace\_id、request\_id 等标识串联一次完整调用
5. 日志的目标是能够复现和定位问题，而不是完整复制网络报文

最终需要取得的是一个平衡，日志中应该保留足够的信息，让工程师能够回答“请求了什么、服务器返回了什么、为什么失败”，但不应该为了排查问题而无限制地记录敏感数据和大体积内容。



---

## 参考

1. https://requests.readthedocs.io/en/latest/user/quickstart/#timeouts
2. https://requests.readthedocs.io/en/latest/user/advanced/#timeouts

