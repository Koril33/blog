---
title: "elk的搭建"
date: 2026-03-03T11:53:40
summary: "ElasticSearch、Logstash、Kibana 在 Linux 的安装和部署"
---

## 前言

ELK 8.x 版本的安装和部署的记录

---

## 安装步骤

### 添加官方源

```shell
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update
```

### 安装

```shell
sudo apt install elasticsearch
sudo apt install kibana
sudo apt install logstash
```

注：elasticsearch 第一安装会输出以下内容：

```
--------------------------- Security autoconfiguration information ------------------------------

Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : WrFx5aWjon-fP5O_VZxo

If this node should join an existing cluster, you can reconfigure this with
'/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
after creating an enrollment token on your existing cluster.

You can complete the following actions at any time:

Reset the password of the elastic built-in superuser with
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.

Generate an enrollment token for Kibana instances with
 '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.

Generate an enrollment token for Elasticsearch nodes with
'/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.

-------------------------------------------------------------------------------------------------
### NOT starting on installation, please execute the following statements to configure elasticsearch service to start automatically using systemd
 sudo systemctl daemon-reload
 sudo systemctl enable elasticsearch.service
### You can start elasticsearch service by executing
 sudo systemctl start elasticsearch.service
```

ElasticSearch 8.X 版本后，默认启用 TLS（后续节点或者API请求走的是 HTTPS），built-in superuser （账号名是：elastic）也就是超级用户的密码是随机生成的，如果忘记了可以通过以下命令重置：

```shell
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

注：网速较慢时，可以给 apt 挂上自己的代理：

```shell
sudo apt -o Acquire::https::Proxy="http://192.168.0.114:7897" install xxx
```

### 修改服务配置

配置文件的路径：

```
/etc/elasticsearch/
/etc/kibana/
/etc/logstash/
```

elasticsearch.yml 默认配置：

```
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
cluster.initial_master_nodes: ["debian-elk"]
http.host: 0.0.0.0
```

默认开启了开启了安全认证，开启了 HTTPS，开启了节点间 TLS，证书路径：/etc/elasticsearch/certs/http_ca.crt

如果仅仅是测试，可以改成简化为单机模式，并且关闭安全配置：

```
discovery.type: single-node
xpack.security.enabled: false
```

kibana.yml 需要修改，添加 elasticsearch 验证的信息：

```
server.host: "0.0.0.0"
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "0D2nWQorTgxZMzhN+R51"
elasticsearch.ssl.verificationMode: none
logging:
  appenders:
    file:
      type: file
      fileName: /var/log/kibana/kibana.log
      layout:
        type: json
  root:
    appenders:
      - default
      - file
pid.file: /run/kibana/kibana.pid
```

可以通过以下命令重置 kibana_system 密码：

```shell
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
```

这里的 elasticsearch.ssl.verificationMode 设置为 none，是因为测试环境，关闭 TLS 验证（生产环境应该开启并且配置好证书）。

---

## 启动

依次启动 ELK 的服务：

```shell

sudo systemctl daemon-reload

sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

sudo systemctl enable kibana
sudo systemctl start kibana

sudo systemctl enable logstash
sudo systemctl start logstash

```

验证是否安装成功：

对于 ElasticSearch 可以用 curl：

```shell
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:WrFx5aWjon-fP5O_VZxo https://localhost:9200
或者
curl -k -u elastic:WrFx5aWjon-fP5O_VZxo https://localhost:9200
{
  "name" : "debian-elk",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "2lffMKvjQCmW_hJ46-YTMQ",
  "version" : {
    "number" : "8.19.12",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "840cd2a58b052d1632219ee0b8dcc0f364226287",
    "build_date" : "2026-02-23T23:08:40.713020893Z",
    "build_snapshot" : false,
    "lucene_version" : "9.12.2",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}

```

`-k`或者`--insecure`表示忽略 TLS 验证，即允许不使用证书访问 SSL 站点。

对于 kibana，只需要打开浏览器，访问服务器IP的 5601 端口，能显示登陆界面就说明安装成功了。
