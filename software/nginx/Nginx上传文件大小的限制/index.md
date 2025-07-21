---
title: "Nginx上传文件大小的限制"
date: 2025-07-21T10:20:00+08:00
summary: "HTTP code 413 的问题"
---

## Nginx 的 413

当向服务器上传文件时，如果碰到 413: Content Too Large，表示请求的 body 大小超过了服务器的限制，如果使用的 web 服务器是 Nginx，那么可以通过一个配置项来解决。

Nginx 有一个配置项叫 client_max_body_size，默认是 1MB，所以在多数文件上传的场景下是不够的。

该参数可以配置在 http 块、server 块或者 location 块，如果希望全局都使用相同的上传文件大小限制的话，那么可以在 http 块中配置：

```nginx
http {
    client_max_body_size 10M;

    server {
        ...
    }
}
```

仅针对某一个站点，则单独在 server 中进行配置：

```nginx
server {
    listen 80;
    server_name git.example.com;

    client_max_body_size 20M;

    location / {
        ...
    }
}
```

仅针对某一个路径，则单独在 location 中进行配置：

```nginx
server {
    listen 80;
    server_name git.example.com;

    location /upload {
        client_max_body_size 300M;
        proxy_pass http://localhost:3000;
    }

    location / {
        client_max_body_size 50M;
    }
}
```

client_max_body_size 必须加单位（如 M、K），否则单位是字节；

建议放在 http 或 server 块中，location 是更细粒度的控制；

修改后需要重启 Nginx：

```sh
# 检查配置是否有误
sudo nginx -t
# 或 restart
sudo systemctl reload nginx
```

---

## 注意

除了 Nginx 会对 request body 大小有做限制之外，其他后端服务框架，比如：Spring Boot 也会对 body 进行限制，所以如果 Nginx 配置好后，仍然 413，则需检查 Nginx 代理的后端服务是否有其他限制。