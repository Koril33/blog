---
title: "CentOS防火墙的一些操作"
date: 2021-05-07T16:23:29+08:00
summary: "CentOS查看防火墙状态，开启，关闭防火墙"
featured_image: "images/background.jpg"
toc: true
---

## 使用systemctl管理firewalld

### 防火墙的状态

```
systemctl status firewalld
```

### 防火墙的开启

```
systemctl start firewalld
```

### 防火墙的关闭

```
systemctl stop firewalld
```

### 设置开机启动防火墙

```
systemctl enable firewalld.service
```

### 设置开机禁用防火墙

```
systemctl disable firewalld.service
```

---

## 使用firewall-cmd进行防火墙的配置

### 查看防火墙状态

```
firewall-cmd --state
```

### 重新加载配置

```
firewall-cmd --reload
```

### 查看开放的端口

```
firewall-cmd --list-ports
```

### 开启防火墙端口

```
firewall-cmd --zone=public --add-port=80/tcp --permanent
```

### 关闭防火墙端口

```
firewall-cmd --zone=public --remove-port=80/tcp --permanent
```

　　命令含义：

　　–zone #作用域

　　–add-port=80/tcp #添加端口，格式为：端口/通讯协议

　　–permanent #永久生效，没有此参数重启后失效

---

## 注意

当我们修改了某些配置之后，firewall并不会立即生效。可以通过两种方式来激活最新配置`systemctl restart firewalld`和`firewall-cmd --reload`两种方式，前一种是重启firewalld服务，建议使用后一种“重载配置文件”。重载配置文件之后不会断掉正在连接的tcp会话，而重启服务则会断开tcp会话。