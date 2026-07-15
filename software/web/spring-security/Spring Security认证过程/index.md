---
title: "Spring Security认证过程"
date: 2026-07-15T10:23:26
summary: "框架如何表示认证对象？认证过程？认证的方式？"
---

## 目录

[TOC]

---

## 前言

应用接收到的认证可能是来自于使用的真人用户，也可能是其他应用系统，认证的凭证也可以多种多样：传统的密码，现代的短信验证码，生物信息（比如指纹），一段加密的 Token。

Spring Security 在认证方面的抽象允许扩展出适用于多种认证体系的过程，本讲简述认证过程中涉及到的各个框架抽象，并且给出一个自定义认证实现。

---

## `Principal` 和 `Authentication`

`Principal` 是 `java.security` 中的一个接口，Spring Security 中的 `Authentication` 继承了此类。

`Principal` 用来表示当前认证的主体，接口有一个 `getName` 方法返回主体的名称：

```java
package java.security;

import javax.security.auth.Subject;

public interface Principal {

    boolean equals(Object another);

    String toString();

    int hashCode();

    String getName();

    default boolean implies(Subject subject) {
        if (subject == null)
            return false;
        return subject.getPrincipals().contains(this);
    }
}
```

和之前提到的 `UserDetails` 不同，`UserDetails` 处于用户名密码体系下的一个更加具体的层次，而 `Principal` 更加通用。`Principal` 经过认证后，可以是 `UserDetails`，也可以表示 JWT 或者匿名用户等。`Principal` 表示当前正在登录或已经登录的那个主体。

`Authentication` 这个接口继承了 `Principal`，在其基础上扩展了凭证、权限、详细信息、是否认证成功等更多内容，它是对一次完整认证过程的抽象。

```java
public interface Authentication extends Principal, Serializable {

	Collection<? extends GrantedAuthority> getAuthorities();
	@Nullable Object getCredentials();
	@Nullable Object getDetails();
	@Nullable Object getPrincipal();
	boolean isAuthenticated();
	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

`Authentication` 既可以表示提交给 `AuthenticationManager` 的待认证请求，又可以表示认证完成后的身份结果。以用户名密码登录的传统认证方式为例：

```
认证前
Authentication {
    principal     = "Alice"
    credentials   = "123456"
    authorities   = []
    authenticated = false
}

认证后
Authentication {
    principal     = UserDetails("Alice") // 从一个字符串变成了 UserDetails 对象
    credentials   = null // 把敏感的凭证信息擦除
    authorities   = [ROLE_USER] // 用户拥有的权限
    authenticated = true // 表示已经通过认证
}
```

---

## `AuthenticationManager`

和 `UserDetails` 需要 `UserDetailsService`、`UserDetailsManager` 一样，`Authentication` 作为主体信息的抽象，需要 `AuthenticationManager` 执行认证过程。

```java
@FunctionalInterface
public interface AuthenticationManager {
	Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

这是一个函数式接口，只有一个 `authenticate` 方法。该方法尝试验证传入的 `Authentication` 对象；如果验证成功，则返回一个完整的 `Authentication` 对象（包括已授予的权限）；如果验证失败，则抛出相应的异常（账户禁用、账户锁定、错误的凭据等）。

---

## `AuthenticationProvider`

一个认证主体可能使用用户名密码、手机短信验证码、JWT 或 API Token，每种体系都是相对独立的认证过程。

如果把认证逻辑堆砌在 `AuthenticationManager` 的实现类中，那么这个实现类会充满 `switch` 或 `if-else` 分支，来对不同的认证体系执行不同的认证过程。

为了解决现实中存在繁多的认证体系，框架提供了 `AuthenticationProvider` 抽象，每一个 `AuthenticationProvider` 实现类封装一种特定类型的认证逻辑。

`AuthenticationProvider` 有两个方法：

```java
public interface AuthenticationProvider {

	@Nullable Authentication authenticate(Authentication authentication) throws AuthenticationException;

	boolean supports(Class<?> authentication);

}
```

### `AuthenticationManager` 和 `AuthenticationProvider` 的关系

它俩的关系是策略模式（Strategy）+ 责任链/委托模式（Chain of Responsibility/Delegation）。

就像之前提到的，现实中有很多认证体系，为了把 `AuthenticationManager` 中的 `if-else` 拆离，就有了 `AuthenticationProvider` 这个抽象。

`AuthenticationManager` 承担的责任是认证调度，找到正确匹配的 `AuthenticationProvider` 并执行认证流程，每个 `AuthenticationProvider` 实现类才负责具体的认证过程。

`AuthenticationManager` 通过 `AuthenticationProvider` 的 `supports` 方法来判断能否把认证请求交给它。

通常一个大型程序包含非常多的认证体系，即对应一系列 `AuthenticationProvider`。`Manager` 会遍历 `Providers`，找到 `supports` 返回 `true` 的那个 `Provider`。

这种责任委托的设计符合开闭原则（对修改关闭，对扩展开放）。增加一个新的认证体系，只需要增加一个新的 `AuthenticationProvider` 实现类，不需要修改 `AuthenticationManager` 的任何逻辑。

---
