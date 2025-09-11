---
title: "Linux命令行设置代理"
date: 2025-09-11T17:58:00+08:00
summary: "设置代理的方法"
---

## 代理

假设你的代理是 http://127.0.0.1:17890，可以在当前终端里执行：

```sh
# 设置 HTTP 代理
export http_proxy="http://127.0.0.1:17890"
export https_proxy="http://127.0.0.1:17890"

# 有些程序需要大写变量名
export HTTP_PROXY="http://127.0.0.1:17890"
export HTTPS_PROXY="http://127.0.0.1:17890"

# 如果有不需要走代理的地址（比如本地仓库）
export no_proxy="localhost,127.0.0.1"
```

验证：

```sh
curl ifconfig.me
```

如果输出的是代理出口 IP，而不是你服务器自身的公网 IP，说明代理生效。

