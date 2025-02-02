---
title: "Docker部署SpringBoot应用"
date: 2023-04-30T16:29:10+08:00
tags: []
featured_image: "images/background.jpg"
summary: "使用 Docker 容器部署 SpringBoot 应用"
toc: true
---

## 前言

本文记录了如何使用 Docker 来部署 SpringBoot 应用。

---

## 一个简单的 SpringBoot 项目

### 简单的 API

为了起步方便，我们暂时先引入 Spring Web 相关的依赖，编写一个 Hello World 级的 controller。

pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.11</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>cn.korilweb</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>demo</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

HelloController.java

```java
package cn.korilweb.demo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello spring boot!";
    }
}
```

### 测试

启动后，使用 curl 测试

```shell
curl.exe http://localhost:8080/hello
# 返回结果
# Hello spring boot!
```

---

## Dockerfile

将打好的 Jar 放到服务器的某个目录上，确保服务器正确安装了 Docker。

dockerfile

```dockerfile
FROM openjdk:8
VOLUME /tmp
ARG JAR_FILE=*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

然后，运行 docker build 命令

```shell
sudo docker build -t koril/my-springboot-app .
```

镜像构建成功

```shell
sudo docker images
# 输出以下内容
REPOSITORY                TAG       IMAGE ID       CREATED          SIZE
koril/my-springboot-app   latest    c5095f16cf03   49 minutes ago   539MB
```

根据刚才构建好的镜像，启动容器

```shell
sudo docker run -dp 80:8080 koril/my-springboot-app
```

查看正在运行的容器

```shell
sudo docker ps
# 输出以下内容
CONTAINER ID   IMAGE                     COMMAND                CREATED          STATUS          PORTS                                   NAMES
9e32ed4dd5c3   koril/my-springboot-app   "java -jar /app.jar"   39 minutes ago   Up 39 minutes   0.0.0.0:80->8080/tcp, :::80->8080/tcp   pedantic_elgamal
```

测试

```shell
curl.exe http://192.168.10.104:80/hello
# 返回结果
Hello spring boot!
```

进入容器

```shell
sudo docker exec -it 9e32ed4dd5c3 /bin/bash
# ls 可以看到根目录下的 app.jar，就是刚刚 COPY 命令执行的结果
# 退出容器，执行 exit 命令即可
```

容器的停止，这里的 9e32ed4dd5c3 就是刚刚启动的容器的 id 值

```shell
sudo docker stop 9e32ed4dd5c3
```

---

## 修改接口

上一节简单介绍了如何在 Docker 容器中启动 SpringBoot，但是如果我想修改 /hello 接口，必须要将修改后的 jar 包，重新上传到服务器，然后重新 build 镜像，再用新的镜像去启动容器。

其实整个开发过程中，只有 jar 包一直在变化，所以可以使用 docker -v 将 Docker 主机下 jar 绑定到容器内部的 jar。

修改 Dockerfile：

```dockerfile
FROM openjdk:8
ENTRYPOINT ["java", "-jar", "/my-app/app.jar"]
```

删除原先的容器和镜像，重新 build，然后启动：

```shell
sudo docker run --name app -dp 80:8080 -v ./package/demo-0.0.1-SNAPSHOT.jar:/my-app/app.jar koril/my-springboot-app
```

我将 Docker 主机当前目录（Dockerfile 所在的目录）下的 package 目录中的 demo-0.0.1-SNAPSHOT.jar 和容器内的 /my-app/app.jar 两个文件绑定在一起，之后修改代码，只需要重新打包，发送到 package 下面，再重启 Docker 容器即可。

```shell
sudo docker restart app
```

---

## 参考

1. https://spring.io/guides/topicals/spring-boot-docker/
2. https://www.baeldung.com/dockerizing-spring-boot-application
