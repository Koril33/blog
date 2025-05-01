---
title: "Debian12安装MongoDB"
date: 2025-02-15T08:58:00+08:00
summary: "Debian12 安装 MongoDB，配置 admin 密码，允许其他主机访问"
---

## 目录

[TOC]

---

## 安装 MongoDB

安装的过程全部参照官方文档：https://www.mongodb.com/zh-cn/docs/manual/tutorial/install-mongodb-on-debian/

### 导入公钥

安装 gnupg 和 curl

```sh
sudo apt install gnupg curl
```

导入公钥

```sh
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
   --dearmor
```

创建 apt 列表

```sh
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] http://repo.mongodb.org/apt/debian bookworm/mongodb-org/8.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

更新 apt

```sh
sudo apt update
```

安装最新的稳定版本

```sh
sudo apt install -y mongodb-org
```

---

## 启动 mongodb

启动

```sh
sudo systemctl start mongod
```

开机自启
```sh
sudo systemctl enable mongod
```

### 碰到的问题

1. 在 win 系统上的 virtualbox 可能没有开启 aux 指令集，导致 mongodb 无法启动，需要关闭 hyper-v，然后把设置中，处理器的特性都勾上。

2. 安装完后，默认是会启动的，但是由于 /var/log/mongodb/mongod.log 权限设置错误，导致无法启动，需要 chmod 和 chown 配置好权限（755 mongodb:mongodb），然后检查 /tmp/mongodb-27017.sock 是否存在，在 restart 之前，需要把这个 sock 文件删掉。

## 设置其他主机能访问

备份配置文件

```sh
sudo cp /etc/mongod.conf /etc/mongod.conf.bak
```

修改 bindIp 字段，改成 0.0.0.0

改好后，保存配置文件，并且重启 mongo

```sh
sudo systemctl restart mongod.service
```

此时，其他主机应该可以连接 mongodb 了，在没有设置用户名和密码前，任何用户都可以连接到该主机，这种状态如果部署在公网环境下，是非常危险的，可以用一个脚本测试下：

```python

import pymongo

def get_mongo_db():
    client = pymongo.MongoClient('192.168.0.122', 27017)
    db = client.study
    return db

def main():
    db = get_mongo_db()
    collection = db['student']

    insert_res = collection.insert_one({
        'name': 'test',
        'age': 22
    })
    print(insert_res.acknowledged)

    find_res = collection.find()
    for item in find_res:
        print(item)


if __name__ == '__main__':
    main()

```

这个脚本仅仅指定了 url 和 port，就连接到了 mongodb，并且创建了数据库和集合。

---

## 添加 admin 用户设置密码

### 添加用户

使用 mongosh 进入 mongodb 交互界面：
```sh

# 选择 admin 数据库
use admin

# 添加用户

db.createUser({user: "koril", pwd: passwordPrompt(), roles: [{role: "userAdminAnyDatabase", db: "admin"}, {role: "readWriteAnyDatabase", db: "admin"}]})

```

### 更改配置文件

配置文件中添加如下内容：

```
security:
  authorization: enabled
```

最后重启 mongod：sudo systemctl restart mongod.service

现在如果再执行上面的 python 脚本，则会爆出下面错误：

```
pymongo.errors.OperationFailure: Command insert requires authentication, full error: {'ok': 0.0, 'errmsg': 'Command insert requires authentication', 'code': 13, 'codeName': 'Unauthorized'}
```

现在需要用户名和密码才能继续访问 mongodb 了。

---

## 参考

1. https://www.jianshu.com/p/f9f1454f251f
2. https://www.mongodb.com/zh-cn/docs/manual/reference/built-in-roles/#std-label-built-in-roles
3. https://www.cnblogs.com/zilongmao/p/11428864.html
4. https://stackoverflow.com/questions/38921414/mongodb-what-are-the-default-user-and-password
