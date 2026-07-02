---
title: "springboot 中 session 的使用"
date: 2026-07-02T11:04:46
summary: ""
---

## 前言

在[这篇博客](https://blog.djhx.site/software/web/backend/cookie%E5%92%8Csession/index.html)中提到了 session 的存储策略，第一种和第二种都是服务器来维护 session 信息。

从服务器的角度来看，就是个 Map，key 是 session id，value 是 session 具体内容。如果 springboot web 默认内置 tomcat，所以不配置额外选项的情况下，使用的是 tomcat 的 session 存储，是保存在 JVM 内存中，意味着断电/更新重启，所有用户 session 会因为没有做持久化而丢失。另外，如果有多个 web 实例做负载均衡，session 没法在不同的进程中共享（也可以做“粘性会话”来解决，同一个用户的请求始终发送到同一个 Tomcat）。

spring session 就是为了解决持久性 session 的问题，提供了一个开箱即用的框架。

---

## Tomcat Session

先来看看不使用 spring session 的情况，默认是由 tomcat 来管理 session，存储在内存中，我们编写三个接口：登录、退出、当前账户，来展示 session 的使用。

```java
@GetMapping("/login")
public String login(HttpServletRequest request) {
    String username = request.getParameter("username");
    // 登录需要一个用户名参数
    if (!StringUtils.hasText(username)) {
        return "username is required";
    }
    // 使用现有，或者创建一个 session
    HttpSession session = request.getSession();
    session.setAttribute("username", username);
    return "login: " + username + "\n";
}


@GetMapping("/logout")
public String logout(HttpServletRequest request) {
	// 退出，需要事先处于登录的状态，也就是存在 session
    HttpSession session = request.getSession(false);
    if (Objects.isNull(session)) {
        return "no session";
    }
    String username = (String) session.getAttribute("username");
    session.invalidate();
    return "logout: " + username + "\n";
}

@GetMapping("/me")
public String me(HttpServletRequest request) {
	// 查看当前用户，需要事先处于登录的状态，也就是存在 session
    HttpSession session = request.getSession(false);
    if (Objects.isNull(session)) {
        return "no session";
    }
    return session.getAttribute("username") + "\n";
}
```

这里需要提一下，getSession 的用法，传递 true 和 false 对于当前 request 没有绑定 session 的情况下的行为是不一样的：

```java
// create 默认是 true
// create 是 true，如果当前 request 没有 session，会创建并返回一个新的 session，如果当前 request 有 session，则返回现有 session
// create 是 false，如果当前 request 没有 session，则会返回 null，如果当前 request 有 session，则返回现有 session
HttpSession getSession(boolean create);
```

大部分情况都填写 false，因为如果选择没有 session 就创建新的 session 返回的话，服务器可能会堆积大量没有用的 session 对象。

理清了 getSession 行为后，考虑在这个 login 接口代码中，下重复登陆的情况，如果 alice 访问了别的接口，是不需要登陆的，但是服务器返回了一个 JSESSIONID，并且由于某些原因，这个 JSESSIONID（未登录）泄露给了黑客。接下来，alice 带着这个 JSESSIONID 请求登录接口，服务器走到这段代码：

```java
// 使用现有，或者创建一个 session
HttpSession session = request.getSession();
session.setAttribute("username", username);
```

会发现 alice 的这个 JSESSIONID 已经存在，就不会创建新的 session 了，用原来的 session 标记为已登录。黑客这个时候就会持有已登录的 JSESSIONID。

防止这种固定会话攻击（Session Fixation）的最简单办法就是调用 changeSessionId，原来的 session 不变，仅仅更换 SESSION ID，把新的 ID 发给 alice，这时候黑客持有的旧的 ID 就失效了：

```java
@GetMapping("/login")
public String login(HttpServletRequest request) {
    String username = request.getParameter("username");
    if (!StringUtils.hasText(username)) {
        return "username is required";
    }

    // HttpSession session = request.getSession();

    // 多一步处理：判断 session 是否存在
    // 存在：changeSessionId
    // 不存在：创建新的 session
    HttpSession session = request.getSession(false);
    if (Objects.isNull(session)) {
        session = request.getSession(true);
    }
    else {
        request.changeSessionId();
    }
	
    session.setAttribute("username", username);

    return "login: " + username + "\n";
}
```

三个接口用 curl 测试：

```

登录

$ curl -i "http://localhost:8080/login?username=alice"
HTTP/1.1 200
Set-Cookie: JSESSIONID=D99E79CBA13A5899A72BEB7F0D78A6A3; Path=/; HttpOnly
Content-Type: text/plain;charset=UTF-8
Content-Length: 13
Date: Thu, 02 Jul 2026 04:00:29 GMT

login: alice

查看当前用户，从上一步的 Set-Cookie 中获取 JSESSIONID

$ curl -i "http://localhost:8080/me" -H "Cookie: JSESSIONID=D99E79CBA13A5899A72BEB7F0D78A6A3"
HTTP/1.1 200
Content-Type: text/plain;charset=UTF-8
Content-Length: 6
Date: Thu, 02 Jul 2026 04:00:54 GMT

alice

退出

$ curl -i "http://localhost:8080/logout" -H "Cookie: JSESSIONID=D99E79CBA13A5899A72BEB7F0D78A6A3"
HTTP/1.1 200
Content-Type: text/plain;charset=UTF-8
Content-Length: 14
Date: Thu, 02 Jul 2026 04:02:01 GMT

logout: alice
```

流程跑通了，接下来展现使用 tomcat session 会碰到的两个主要问题：

1. 服务重启后，原来的用户 session 丢失
2. 负载均衡，用户一会处于登录，一会处于没有登陆的状态

### 辅助接口

因为 session 本质是存在一个 ConcurrentHashMap 里面的：

```java
protected Map<String,Session> sessions = new ConcurrentHashMap<>();
```

想要看到当前登录用户列表和根据 JSESSIONID 拿到对应的 session 信息，需要一些额外的技巧。

这里提供两个辅助接口，可以直观的看到当前系统的 session 信息：

```java

@GetMapping("/online-users")
public List<Map<String, Object>> users(HttpServletRequest request) {
    Manager manager = tomcatManager(request);

    return Arrays.stream(manager.findSessions())
            .filter(Session::isValid)
            .map(this::sessionInfo)
            .toList();
}

@GetMapping("/session/{id}")
public Object session(@PathVariable String id, HttpServletRequest request) throws IOException {
    Manager manager = tomcatManager(request);
    Session session = manager.findSession(id);

    if (session == null || !session.isValid()) {
        return "session not found: " + id;
    }

    return sessionInfo(session);
}

private Manager tomcatManager(HttpServletRequest request) {
    Request tomcatRequest = tomcatRequest(request);
    Manager manager = tomcatRequest.getContext().getManager();

    if (manager == null) {
        throw new IllegalStateException("Tomcat Manager not found");
    }

    return manager;
}

private Request tomcatRequest(HttpServletRequest request) {
    ServletRequest current = request;

    while (current instanceof HttpServletRequestWrapper wrapper) {
        current = wrapper.getRequest();
    }

    if (current instanceof RequestFacade requestFacade) {
        return (Request) readField(requestFacade, "request");
    }

    throw new IllegalStateException("Current request is not Tomcat RequestFacade: " + current.getClass());
}

private Object readField(Object target, String fieldName) {
    try {
        Field field = target.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        return field.get(target);
    } catch (ReflectiveOperationException e) {
        throw new IllegalStateException("Failed to read field: " + fieldName, e);
    }
}

private Map<String, Object> sessionInfo(Session session) {
    Map<String, Object> result = new LinkedHashMap<>();
    HttpSession httpSession = session.getSession();

    result.put("id", session.getId());
    result.put("valid", session.isValid());
    result.put("new", session.isNew());
    result.put("authType", session.getAuthType());
    result.put("principal", session.getPrincipal() == null ? null : session.getPrincipal().getName());
    result.put("creationTime", Instant.ofEpochMilli(session.getCreationTime()).toString());
    result.put("thisAccessedTime", Instant.ofEpochMilli(session.getThisAccessedTime()).toString());
    result.put("lastAccessedTime", Instant.ofEpochMilli(session.getLastAccessedTime()).toString());
    result.put("idleTimeMillis", session.getIdleTime());
    result.put("maxInactiveIntervalSeconds", session.getMaxInactiveInterval());

    Map<String, Object> attributes = new LinkedHashMap<>();
    Enumeration<String> names = httpSession.getAttributeNames();
    while (names.hasMoreElements()) {
        String name = names.nextElement();
        Object value = httpSession.getAttribute(name);

        Map<String, Object> attribute = new LinkedHashMap<>();
        attribute.put("type", value == null ? null : value.getClass().getName());
        attribute.put("value", value == null ? null : String.valueOf(value));

        attributes.put(name, attribute);
    }

    result.put("attributes", attributes);
    return result;
}
```

### 服务重启

假设现在 alice 登录了后台：

```shell
$ curl -i "http://localhost:8080/login?username=alice"
HTTP/1.1 200
Set-Cookie: JSESSIONID=41C6050C481315A1AF57E5BC6D81787B; Path=/; HttpOnly
Content-Type: text/plain;charset=UTF-8
Content-Length: 13
Date: Thu, 02 Jul 2026 08:16:26 GMT

login: alice
```

拿到了 JSESSIONID，通过 /online-users 接口可以看到：

```shell
$ curl "http://localhost:8080/online-users"
[{"id":"41C6050C481315A1AF57E5BC6D81787B","valid":true,"new":false,"authType":null,"principal":null,"creationTime":"2026-07-02T08:16:26.794Z","thisAccessedTime":"2026-07-02T08:16:26.798Z","lastAccessedTime":"2026-07-02T08:16:26.798Z","idleTimeMillis":66386,"maxInactiveIntervalSeconds":120,"attributes":{"username":{"type":"java.lang.String","value":"alice"}}}]
```

list 中有一个对象，就是 alice 的 session，如果重启了 SpringBoot 项目，alice 拿到的 JSESSIONID 就没有用了。

### 负载均衡

启动两个 springboot： 

```shell
java -jar App.jar --server.port=8080

java -jar App.jar --server.port=8081
```

然后 nginx 简单配置一下负载均衡：

```nginx
upstream backend {
    server localhost:8080;
    server localhost:8081;
}

server {
    listen 8888;

    location / {
        proxy_pass http://backend;
    }
}
```

两个 tomcat 的 session 内存各自独立，所以 alice 登陆成功后，session 存在其中一台机器上，另一台机器无法访问到这个 session，就会出现一会登录，一会没有登录的情况。

---

## Spring Session

HttpSession 是 Servlet 标准提供的服务端会话接口：

```java
HttpSession session = request.getSession();
session.setAttribute("username", "djhx");
```

提供了一系列标准操作：

```java
session.getId();
session.setAttribute("username", username);
session.getAttribute("username");
session.removeAttribute("username");
session.invalidate();
```

但它仅仅是接口，不同的实现可以把 session 存储在不同的位置，比如 tomcat 内存，redis，关系型数据库等等。

Spring Session 没有修改 HttpSession 接口，而是通过 Filter 包装请求，让 request.getSession() 返回由 Spring Session 管理的 HttpSession 实现。

引入 Spring Session 以后，我们的接口可以不变，使用 session 的方式也没有改变，这就是依赖于抽象的好处（随时可以替换掉底层实现，而不必改动代码）。

### 依赖和配置

引入一个 starter 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-session-data-redis</artifactId>
</dependency>
```

配置项：

```properties
spring.data.redis.host=127.0.0.1
spring.data.redis.port=6379
spring.data.redis.database=0
spring.data.redis.password=123456
```

重新启动，三个接口完全不用修改，就可以使用 Spring Session 了。

### 测试

登录：

```shell

$ curl -i "http://192.168.0.200:8888/login?username=alice"
HTTP/1.1 200
Server: nginx/1.22.1
Date: Thu, 02 Jul 2026 08:40:45 GMT
Content-Type: text/plain;charset=UTF-8
Content-Length: 13
Connection: keep-alive
Set-Cookie: SESSION=NTZjNWZkZjgtYmRlMS00MTUzLTliYTYtNmZjNmQ1YzJkYjEx; Path=/; HttpOnly; SameSite=Lax

login: alice

```

spring session 的 cookie key 是 SESSION 区别于 Tomcat JSESSIONID，登陆后可以看到 redis 里面多了个`spring:session:sessions:56c5fdf8-bde1-4153-9ba6-6fc6d5c2db11`的 Hash。

因为 session 持久化在了 redis，负载均衡后的两台机器共同连接一个 redis，所以重启应用导致 session 丢失，还是负载均衡的问题，都得以解决。

---

## 序列化

spring session 默认使用 java 序列化，所以 redis 存储的 session 信息是二进制的，可以修改默认序列化使用 Jackson：

```java

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.serializer.GenericJacksonJsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializer;
import tools.jackson.databind.jsontype.BasicPolymorphicTypeValidator;

@Configuration
public class SessionSerializationConfig {

    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        var typeValidator = BasicPolymorphicTypeValidator.builder()
                // 替换成你的实体类所在包
                .allowIfSubType("site.djhx.studyspringboot.entity")
                // 如果 Session 中需要保存 List、Map 等集合
                .allowIfSubType("java.util.")
                .build();

        return GenericJacksonJsonRedisSerializer.builder()
                .enableDefaultTyping(typeValidator)
                .typePropertyName("@class")
                .build();
    }
}
```

---

## 参考

1. https://docs.spring.io/spring-session/reference/guides/boot-redis.html
2. https://docs.spring.io/spring-session/reference/api/java/org/springframework/session/data/redis/RedisIndexedSessionRepository.html
3. https://docs.spring.io/spring-session/reference/4.1-SNAPSHOT/configuration/redis.html