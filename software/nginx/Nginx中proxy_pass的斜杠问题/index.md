---
title: "Nginx中proxy_pass的斜杠问题"
date: 2023-12-23T11:38:33+08:00
tags: []
featured_image: "images/background.jpg"
summary: "proxy_pass 和 location 的末尾斜杠问题"
toc: true
---

## 前言

在 Nginx 的日常使用中，经常需要反向代理一些接口，而接口和 location 的末尾斜杠和匹配问题，由于可能性太多，常常弄混，故写此文记录之。

---

## proxy_pass 不带目录

后端接口：http://192.168.0.180:8080/api/default

前端请求：http://192.168.0.180:80/demo/api/default

### 第一种

location 末尾不带斜杠，proxy_pass 末尾不带斜杠

nginx 配置

```nginx
server {
    listen 80;

    location /demo {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

实际访问：http://192.168.0.180:8080/demo/api/default

结论：location 和 proxy_pass 都不带斜杠，实际访问地址 = proxy_pass + location + /api/default

### 第二种

location 末尾带斜杠，proxy_pass 末尾不带斜杠

nginx 配置

```nginx
server {
    listen 80;

    location /demo/ {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

实际访问：http://192.168.0.180:8080/demo/api/default

结论：和第一种情况一致。

### 第三种

location 末尾不带斜杠，proxy_pass 末尾带斜杠

nginx 配置

```nginx
server {
    listen 80;

    location /demo {
        proxy_pass http://127.0.0.1:8080/;
    }
}
```

实际访问：http://192.168.0.180:8080//api/default

结论：proxy_pass 末尾带斜杠时，就不会拼接 location 的代理路径了，但是会多出来一个斜杠，实际访问地址 = proxy_pass + /api/default，由于 proxy_pass 末尾带一个斜杠，而 /api/default 又以斜杠开头，导致中间存在两个斜杠。

### 第四种

location 末尾带斜杠，proxy_pass 末尾带斜杠

nginx 配置

```nginx
server {
    listen 80;

    location /demo/ {
        proxy_pass http://127.0.0.1:8080/;
    }
}
```

实际访问：http://192.168.0.180:8080/api/default

结论：只有第四种，能够正常访问到后端接口，因为 proxy_pass 末尾带斜杠，所以不会拼接 location 的代理路径，并且 /demo/api/default 由于 location 是 /demo/，所以 api 前面的 / 被去掉了，接口变成 api/default，就不会出现第三种方案中的两个斜杠的问题，实际访问地址 = proxy_pass + api/default

表格

| location |       proxy_pass       |               前端访问路径               |                实际访问路径                | 结果 |
| :------: | :--------------------: | :--------------------------------------: | :----------------------------------------: | :--: |
|  /demo   | http://127.0.0.1:8080  | http://192.168.0.180:80/demo/api/default | http://192.168.0.180:8080/demo/api/default | 失败 |
|  /demo/  | http://127.0.0.1:8080  | http://192.168.0.180:80/demo/api/default | http://192.168.0.180:8080/demo/api/default | 失败 |
|  /demo   | http://127.0.0.1:8080/ | http://192.168.0.180:80/demo/api/default |   http://192.168.0.180:8080//api/default   | 失败 |
|  /demo/  | http://127.0.0.1:8080/ | http://192.168.0.180:80/demo/api/default |   http://192.168.0.180:8080/api/default    | 成功 |

---

## proxy_pass 带目录

后端接口：http://192.168.0.180:8080/api/default

前端请求：http://192.168.0.180:80/demo/api/default

### 第五种

location 末尾不带斜杠，proxy_pass 末尾不带斜杠

nginx 配置

```nginx
server {
    listen 80;

    location /demo {
        proxy_pass http://127.0.0.1:8080/api;
    }
}
```

实际访问：http://192.168.0.180:8080/api/api/default

结论：和第一种不一样，尽管 location 和 proxy_pass 末尾都不带斜杠，但是 location 的代理路径并不会拼接到 proxy_pass 后面，实际访问地址 = proxy_pass + /api/default。

### 第六种

location 末尾带斜杠，proxy_pass 末尾不带斜杠

nginx 配置

```nginx
server {
    listen 80;

    location /demo/ {
        proxy_pass http://127.0.0.1:8080/api;
    }
}
```

实际访问：http://192.168.0.180:8080/apiapi/default

结论：和第五种很相似，不过由于 location 末尾带了斜杠将 /api/default 前面的斜杠给去掉了，实际访问路径 = proxy_pass + api/default。

### 第七种

location 末尾不带斜杠，proxy_pass 末尾带斜杠

nginx 配置

```nginx
server {
    listen 80;

    location /demo {
        proxy_pass http://127.0.0.1:8080/api/;
    }
}
```

实际访问：http://192.168.0.180:8080/api//api/default

结论：和第五种很相似，不过由于 proxy_pass 末尾带了斜杠，所以导致出现了第三种情况类似的问题，有两个重复的斜杠，实际访问路径 = proxy_pass + /api/default。

### 第八种

location 末尾带斜杠，proxy_pass 末尾带斜杠

nginx 配置

```nginx
server {
    listen 80;

    location /demo/ {
        proxy_pass http://127.0.0.1:8080/api/;
    }
}
```

实际访问：http://192.168.0.180:8080/api/api/default

结论：和第五种的结果一致，实际访问路径 = proxy_pass + api/default。

表格

| location |         proxy_pass         |               前端访问路径               |                实际访问路径                | 结果 |
| :------: | :------------------------: | :--------------------------------------: | :----------------------------------------: | :--: |
|  /demo   | http://127.0.0.1:8080/api  | http://192.168.0.180:80/demo/api/default | http://192.168.0.180:8080/api/api/default  | 失败 |
|  /demo/  | http://127.0.0.1:8080/api  | http://192.168.0.180:80/demo/api/default |  http://192.168.0.180:8080/apiapi/default  | 失败 |
|  /demo   | http://127.0.0.1:8080/api/ | http://192.168.0.180:80/demo/api/default | http://192.168.0.180:8080/api//api/default | 失败 |
|  /demo/  | http://127.0.0.1:8080/api/ | http://192.168.0.180:80/demo/api/default | http://192.168.0.180:8080/api/api/default  | 失败 |

---

## proxy_pass 总结

proxy_pass 的参数分为以下三类：

1. http://ip:port
2. http://ip:port/
3. http://ip:port/dir

如果按照 port 后面是否还跟着字符串来分，第二类和第三类可以合并成一类：

1. port 后面不跟任何字符串
2. port 后面还跟着字符串

对于第一类，实际访问路径 = proxy_pass + 匹配的代理路径

对于第二类，实际访问路径 = proxy_pass + 匹配的代理路径（删除 location 参数）

---

## 参考

1. https://blog.csdn.net/weishuai90/article/details/131133621
2. https://blog.csdn.net/s827292890/article/details/129583746
