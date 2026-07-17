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

## ProviderManager

`ProviderManager` 是 `AuthenticationManager` 的一个实现类。

`ProviderManager` 按顺序询问多个 Provider，成功就停止，普通的失败先记下来并继续，账号状态异常或者 provider 内部异常立即停止，没有 provider 能处理就询问父级 provider，最终仍没人处理则抛出 ProviderNotFoundException。

核心逻辑：

- provider.supports() 为 false：跳过。
- provider.authenticate() 返回 null：继续下一个 Provider。
- provider.authenticate() 抛 AuthenticationException：记录异常，继续下一个 Provider。
- provider.authenticate() 抛 AccountStatusException / InternalAuthenticationServiceException：立即终止认证。
- provider.authenticate() 返回 Authentication：立即成功，不再尝试后续 Provider。


源码和注释：

```java

@Override
public Authentication authenticate(Authentication authentication) throws AuthenticationException {

    // 获取待认证的 Authentication 的 Class
    Class<? extends Authentication> toTest = authentication.getClass();
    
    // 记录最近一次 AuthenticationException，
    // 如果所有 provider 都失败，最终会重新抛出该异常
    AuthenticationException lastException = null;
    
    // parent provider 抛出的异常
    AuthenticationException parentException = null;
    
    // 当前的 provider 验证结果
    Authentication result = null;
    
    // parent provider 验证结果
    Authentication parentResult = null;

    // 计数器，当前第几个支持该 Authentication 的 Provider
    int currentPosition = 0;
    int size = this.providers.size();

    // 遍历 ProviderManager 中的所有 providers
    for (AuthenticationProvider provider : getProviders()) {

        // 判断当前 provider 是否能够处理这种 Authentication 类型。
        // supports() 仅表示 provider "能够处理"，并不表示认证成功。
        if (!provider.supports(toTest)) {
            continue;
        }

        // Trace 日志
        if (logger.isTraceEnabled()) {
            logger.trace(LogMessage.format("Authenticating request with %s (%d/%d)",
                    provider.getClass().getSimpleName(), ++currentPosition, size));
        }

        // try 块中尝试使用 provider 进行认证
        try {
            result = provider.authenticate(authentication);

            // provider 的 authentication 可以返回 null，返回 null 的语义是：当前 provider 放弃处理，让下一个 provider 尝试。
            if (result != null) {
                // 将原 Authentication 中的 details（例如 IP、Session 信息）复制到认证成功后的 Authentication。
                copyDetails(authentication, result);
                // 认证成功，退出循环
                break;
            }
        }
        // 碰到 AccountStatusException 说明当前账号状态有问题，不能再继续尝试别的 provider 重新抛出异常
        // AccountStatusException 包括：账号禁用，账号锁定，账号过期，凭证过期等，再尝试其它 Provider 已经没有意义
        catch (AccountStatusException ex) {
            prepareException(ex, authentication);
            logger.debug(LogMessage.format("Authentication failed for user '%s' since their account status is %s",
                    authentication.getName(), ex.getMessage()), ex);
            // SEC-546: Avoid polling additional providers if auth failure is due to
            // invalid account status
            throw ex;
        }
        // 认证服务内部已经发生错误，不应该再继续尝试其它 Provider
        catch (InternalAuthenticationServiceException ex) {
            prepareException(ex, authentication);
            logger.debug(LogMessage.format("Authentication service failed internally for user '%s'",
                    authentication.getName()), ex);
            // SEC-546: Avoid polling additional providers if auth failure is due to
            // invalid account status
            throw ex;
        }
        // 如果是 AuthenticationException 记录异常，尝试下一个 provider
        catch (AuthenticationException ex) {
            ex.setAuthenticationRequest(authentication);
            logger.debug(LogMessage.format("Authentication failed with provider %s since %s",
                    provider.getClass().getSimpleName(), ex.getMessage()));
            lastException = ex;
        }
    }
    
    // 当前 ProviderManager 未认证成功，则委托 parent AuthenticationManager 再尝试认证。
    // 无论是所有 provider 返回 null，还是均认证失败，都会尝试 parent
    if (result == null && this.parent != null) {
        // Allow the parent to try.
        try {
            parentResult = this.parent.authenticate(authentication);
            result = parentResult;
        }
        catch (ProviderNotFoundException ex) {
            // ignore as we will throw below if no other exception occurred prior to
            // calling parent and the parent
            // may throw ProviderNotFound even though a provider in the child already
            // handled the request
        }
        catch (AuthenticationException ex) {
            parentException = ex;
            lastException = ex;
        }
    }
    if (result != null) {
        // 擦除敏感凭证信息
        if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
            // Authentication is complete. Remove credentials and other secret data
            // from authentication
            ((CredentialsContainer) result).eraseCredentials();
        }
        // If the parent AuthenticationManager was attempted and successful then it
        // will publish an AuthenticationSuccessEvent
        // This check prevents a duplicate AuthenticationSuccessEvent if the parent
        // AuthenticationManager already published it
        if (parentResult == null) {
            this.eventPublisher.publishAuthenticationSuccess(result);
        }

        return result;
    }

    // Parent was null, or didn't authenticate (or throw an exception).
    // 如果没有任何 provider 能处理该 Authentication 类型，
    // 抛出 ProviderNotFoundException
    if (lastException == null) {
        lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound",
                new Object[] { toTest.getName() }, "No AuthenticationProvider found for {0}"));
    }
    // If the parent AuthenticationManager was attempted and failed then it will
    // publish an AbstractAuthenticationFailureEvent
    // This check prevents a duplicate AbstractAuthenticationFailureEvent if the
    // parent AuthenticationManager already published it
    if (parentException == null) {
        prepareException(lastException, authentication);
    }

    // Ensure this message is not logged when authentication is attempted by
    // the parent provider
    if (this.parent != null) {
        logger.debug("Denying authentication since all attempted providers failed");
    }

    // 抛出最后一次异常信息
    throw lastException;
}

```

## DaoAuthenticationProvider

`DaoAuthenticationProvider` 是 `AuthenticationProvider` 的一个重要的实现类，是用户名密码认证体系下真正实现认证的 provider。

继承关系：

```

AuthenticationProvider
        ▲
AbstractUserDetailsAuthenticationProvider
        ▲
DaoAuthenticationProvider

```

`AuthenticationProvider` 接口只提供了 supports 和 authenticate 两个方法，而 `AbstractUserDetailsAuthenticationProvider` 这个抽象类实现了大部分的模板代码：

- 查询缓存
- 调用 UserDetailsService
- 检查账号状态
- 创建 Authentication
- 用户缓存(UserCache)

`AbstractUserDetailsAuthenticationProvider` 真正留给子类需要实现的只有两个方法：

```java

protected abstract void additionalAuthenticationChecks(
        UserDetails userDetails,
        UsernamePasswordAuthenticationToken authentication);

protected abstract UserDetails retrieveUser(
        String username,
        UsernamePasswordAuthenticationToken authentication);

```

首先看 supports，`AbstractUserDetailsAuthenticationProvider` 的 supports 如下：

```java

@Override
public boolean supports(Class<?> authentication) {
    return (UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));
}

```

表明它的子类，仅支持 `UsernamePasswordAuthenticationToken` 这种类型的 `Authentication`。

authenticate 的逻辑也在这个抽象类之中：

```java

@Override
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
            () -> this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.onlySupports",
                    "Only UsernamePasswordAuthenticationToken is supported"));

    // 从 Authentication 中提取用户名。
    // 默认返回 authentication.getName()，子类也可以通过 determineUsername()
    // 自定义用户名的提取方式。
    String username = determineUsername(authentication);
    boolean cacheWasUsed = true;

    // 优先从 UserCache 中获取 UserDetails。
    // UserCache 缓存的是用户信息（UserDetails），而不是 Authentication。
    // 默认使用 NullUserCache，即默认不开启缓存。
    UserDetails user = this.userCache.getUserFromCache(username);

    // 缓存中没有
    if (user == null) {
        cacheWasUsed = false;
        try {
            // 调用子类实现 retrieveUser() 获取 UserDetails。
            // AbstractUserDetailsAuthenticationProvider 不关心用户来自哪里，
            // 可以来自数据库、LDAP、远程服务等。
            // DaoAuthenticationProvider 的实现就是调用 UserDetailsService.loadUserByUsername()。
            user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
        }
        catch (UsernameNotFoundException ex) {
            this.logger.debug(LogMessage.format("Failed to find user '%s'", username));
            String message = this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials",
                    "Bad credentials");
            if (!this.hideUserNotFoundExceptions) {
                throw ex;
            }

            // 默认隐藏 UsernameNotFoundException，统一转换为 BadCredentialsException，
            // 避免攻击者根据异常类型判断用户名是否存在。
            throw new BadCredentialsException(message, ex);
        }
        Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");
    }

    // 认证前检查账号状态。
    // 默认检查：
    //  - enabled
    //  - accountNonLocked
    //  - accountNonExpired
    // 检查失败分别抛出 DisabledException、LockedException、AccountExpiredException。
    try {
        performPreCheck(user, (UsernamePasswordAuthenticationToken) authentication);
    }
    catch (AuthenticationException ex) {
        if (!cacheWasUsed) {
            throw ex;
        }
        // There was a problem, so try again after checking
        // we're using latest data (i.e. not from the cache)
        cacheWasUsed = false;
        user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
        performPreCheck(user, (UsernamePasswordAuthenticationToken) authentication);
    }

    // 检查凭证是否过期
    this.postAuthenticationChecks.check(user);
    
    // 如果当前 UserDetails 来自缓存，并且账号状态检查失败，
    // 则重新从数据源加载最新的 UserDetails 再检查一次。
    // 避免由于缓存过期导致认证失败。
    if (!cacheWasUsed) {
        this.userCache.putUserInCache(user);
    }
    Object principalToReturn = user;
    if (this.forcePrincipalAsString) {
        principalToReturn = user.getUsername();
    }
    // 认证成功，返回 Authentication
    return createSuccessAuthentication(principalToReturn, authentication, user);
}


private void performPreCheck(UserDetails user, UsernamePasswordAuthenticationToken authentication) {
    try {
        this.preAuthenticationChecks.check(user);
    }
    catch (AuthenticationException ex) {
        if (!this.alwaysPerformAdditionalChecksOnUser) {
            throw ex;
        }
        try {
            additionalAuthenticationChecks(user, authentication);
        }
        catch (AuthenticationException ignored) {
            // preserve the original failed check
        }
        throw ex;
    }

    // 调用子类完成额外认证逻辑。
    // 对 DaoAuthenticationProvider 来说，这一步负责校验用户输入密码
    // 是否与 UserDetails 中保存的密码匹配。
    additionalAuthenticationChecks(user, authentication);
}

```

`AbstractUserDetailsAuthenticationProvider` 的职责是定义了基于 `UserDetails` 的认证模板流程，而把“如何查询用户”和“如何校验凭证”交给子类实现。

`DaoAuthenticationProvider` 填充了模板中的两个关键扩展点：

retrieveUser()：调用 UserDetailsService.loadUserByUsername() 获取用户；
additionalAuthenticationChecks()：使用 PasswordEncoder.matches() 校验密码。

源码如下：

```java

@Override
protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication)
        throws AuthenticationException {
    prepareTimingAttackProtection();
    try {
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        if (loadedUser == null) {
            throw new InternalAuthenticationServiceException(
                    "UserDetailsService returned null, which is an interface contract violation");
        }
        return loadedUser;
    }
    catch (UsernameNotFoundException ex) {
        mitigateAgainstTimingAttack(authentication);
        throw ex;
    }
    catch (InternalAuthenticationServiceException ex) {
        throw ex;
    }
    catch (Exception ex) {
        throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
    }
}


@Override
protected void additionalAuthenticationChecks(UserDetails userDetails,
        UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
    if (authentication.getCredentials() == null) {
        this.logger.debug("Failed to authenticate since no credentials provided");
        throw new BadCredentialsException(this.messages
            .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
    }
    // 密码校验
    String presentedPassword = authentication.getCredentials().toString();
    if (!this.passwordEncoder.get().matches(presentedPassword, userDetails.getPassword())) {
        this.logger.debug("Failed to authenticate since password does not match stored value");
        throw new BadCredentialsException(this.messages
            .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
    }
}


```

---

## SecurityContext

Context（上下文）的本质是计算机在某个环境或某个执行过程中，需要保存和共享的一些状态信息数据。

比如两个人在对话，你作为第三个人加入，你并不知道之前他们聊了什么，所以听到的内容一知半解，这就是缺乏他们俩的对话上下文导致的。

SecurityContext 就是当前安全信息的上下文，里面保存的是 Authentication：

```java

public interface SecurityContext extends Serializable {
    @Nullable Authentication getAuthentication();
    void setAuthentication(@Nullable Authentication authentication);
}

```

SecurityContext 接口定义了与当前执行线程关联的最低安全信息的接口。

---

## SecurityContextHolderFilter

在 HttpSecurityConfiguration 注入的 HttpSecurity Bean 中，有一个配置是关于 SecurityContext 的：

```java

@Bean(HTTPSECURITY_BEAN_NAME)
    @Scope("prototype")
    HttpSecurity httpSecurity() {
        LazyPasswordEncoder passwordEncoder = new LazyPasswordEncoder(this.context);
        AuthenticationManagerBuilder authenticationBuilder = new DefaultPasswordEncoderAuthenticationManagerBuilder(
                this.objectPostProcessor, passwordEncoder);
        authenticationBuilder.parentAuthenticationManager(authenticationManager());
        authenticationBuilder.authenticationEventPublisher(getAuthenticationEventPublisher());
        HttpSecurity http = new HttpSecurity(this.objectPostProcessor, authenticationBuilder, createSharedObjects());
        WebAsyncManagerIntegrationFilter webAsyncManagerIntegrationFilter = new WebAsyncManagerIntegrationFilter();
        webAsyncManagerIntegrationFilter.setSecurityContextHolderStrategy(this.securityContextHolderStrategy);
        // @formatter:off
        http
            .csrf(withDefaults())
            .addFilter(webAsyncManagerIntegrationFilter)
            .exceptionHandling(withDefaults())
            .headers(withDefaults())
            .sessionManagement(withDefaults())
            .securityContext(withDefaults())
            .requestCache(withDefaults())
            .anonymous(withDefaults())
            .servletApi(withDefaults())
            .with(new DefaultLoginPageConfigurer<>());
        http.logout(withDefaults());
        // @formatter:on
        applyCorsIfAvailable(http);
        applyDefaultConfigurers(http);
        applyHttpSecurityCustomizers(this.context, http);
        applyTopLevelCustomizers(this.context, http);
        return http;
    }

```

默认配置了 .securityContext(withDefaults())：

```java

public HttpSecurity securityContext(Customizer<SecurityContextConfigurer<HttpSecurity>> securityContextCustomizer) {
    securityContextCustomizer.customize(getOrApply(new SecurityContextConfigurer<>()));
    return HttpSecurity.this;
}

```

configure 如下：

```java

@Override
@SuppressWarnings("unchecked")
public void configure(H http) {
    SecurityContextRepository securityContextRepository = getSecurityContextRepository();
    if (this.requireExplicitSave) {
        SecurityContextHolderFilter securityContextHolderFilter = postProcess(
                new SecurityContextHolderFilter(securityContextRepository));
        securityContextHolderFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());
        http.addFilter(securityContextHolderFilter);
    }
    else {
        SecurityContextPersistenceFilter securityContextFilter = new SecurityContextPersistenceFilter(
                securityContextRepository);
        securityContextFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());
        SessionManagementConfigurer<?> sessionManagement = http.getConfigurer(SessionManagementConfigurer.class);
        SessionCreationPolicy sessionCreationPolicy = (sessionManagement != null)
                ? sessionManagement.getSessionCreationPolicy() : null;
        if (SessionCreationPolicy.ALWAYS == sessionCreationPolicy) {
            securityContextFilter.setForceEagerSessionCreation(true);
            http.addFilter(postProcess(new ForceEagerSessionCreationFilter()));
        }
        securityContextFilter = postProcess(securityContextFilter);
        http.addFilter(securityContextFilter);
    }
}

```

该 configurer 引入了 SecurityContextHolderFilter：

```java


private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws ServletException, IOException {
    if (request.getAttribute(FILTER_APPLIED) != null) {
        chain.doFilter(request, response);
        return;
    }
    request.setAttribute(FILTER_APPLIED, Boolean.TRUE);

    // 从 SecurityContextRepository 延迟获取 SecurityContext
    Supplier<SecurityContext> deferredContext = this.securityContextRepository.loadDeferredContext(request);

    try {
        this.securityContextHolderStrategy.setDeferredContext(deferredContext);
        chain.doFilter(request, response);
    }
    finally {        
        // 清理当前线程的 SecurityContext，因为请求结束后，线程会重新回到线程池
        // 避免下一个请求获取到当前请求的 SecurityContext，请求结束必须清理上下文信息
        this.securityContextHolderStrategy.clearContext();
        request.removeAttribute(FILTER_APPLIED);
    }
}

```

SecurityContextHolderFilter 这个过滤器的作用就是负责让一次 HTTP 请求拥有自己的 SecurityContext，并绑定到 SecurityContextHolder，请求结束后再解绑。

关于如何让请求拥有 SecurityContext 以及如何存储，需要引出 Tomcat、Servlet、线程池与 Spring Security 认证体系的关联。

---

## 上下文存储

重新回到 Tomcat 和 Servlet 的世界，一个请求进入了 Tomcat 容器，Tomcat 会从线程池中选择一个空闲线程来处理该请求，这个请求进入了 Spring Security 的 SecurityFilterChain，也就是一连串的过滤器，经过某个 AuthenticationFilter 捕获了请求中的认证凭证信息，交给了 AuthenticationManager 去认证，如果认证成功了，AuthenticationManager 会返回给 filter 一个认证成功后的 Authentication 对象（里面包含了用户信息，拥有的权限），那么问题是这个认证成功后的 Authentication 如何存储到上下文环境中，以供后面的 filter 或者 controller 访问呢？

对于 HTTP 请求而言，有两种需要访问到 Authentication 的情况：

1. 当前线程存储：一个用户发起的一个 HTTP 访问，当前线程在认证成功后的后续步骤，需要能访问到用户信息。
2. 跨请求存储：同一个用户发起的不同 HTTP 访问，会经由不同线程处理，不同线程能访问到同一个用户信息。

### 当前线程-SecurityContextHolderStrategy

Spring Security 对于当前线程存储 SecurityContext 的默认方式是 ThreadLocal。

无论是 Session 模式还是 Stateless 模式，都需要 SecurityContextHolder 来保存凭证信息，以供请求在当前线程的后续访问。

而 SecurityContextHolder 保存 SecurityContext 会有多种方式，所以衍生出了 SecurityContextHolderStrategy 这个接口，这又

### 跨请求-SecurityContextRepository

Spring Security 对于跨请求的访问凭证恢复和存储，使用的方案是 Cookie/Session 机制。

在 HttpSecurityConfiguration 中：

```java

http
    .csrf(withDefaults())
    .addFilter(webAsyncManagerIntegrationFilter)
    .exceptionHandling(withDefaults())
    .headers(withDefaults())
    .sessionManagement(withDefaults())
    .securityContext(withDefaults())
    .requestCache(withDefaults())
    .anonymous(withDefaults())
    .servletApi(withDefaults())
    .with(new DefaultLoginPageConfigurer<>());

```

配置 securityContext 前，还配置了 sessionManagement，SessionManagementConfigurer 在 init 方法中会引入 SecurityContextRepository：

```java

@Override
public void init(H http) {
    SecurityContextRepository securityContextRepository = http.getSharedObject(SecurityContextRepository.class);
    boolean stateless = isStateless();
    if (securityContextRepository == null) {
        if (stateless) {
            http.setSharedObject(SecurityContextRepository.class, new RequestAttributeSecurityContextRepository());
            this.sessionManagementSecurityContextRepository = new NullSecurityContextRepository();
        }
        else {
            HttpSessionSecurityContextRepository httpSecurityRepository = new HttpSessionSecurityContextRepository();
            httpSecurityRepository.setDisableUrlRewriting(!this.enableSessionUrlRewriting);
            httpSecurityRepository.setAllowSessionCreation(isAllowSessionCreation());
            AuthenticationTrustResolver trustResolver = http.getSharedObject(AuthenticationTrustResolver.class);
            if (trustResolver != null) {
                httpSecurityRepository.setTrustResolver(trustResolver);
            }
            this.sessionManagementSecurityContextRepository = httpSecurityRepository;
            DelegatingSecurityContextRepository defaultRepository = new DelegatingSecurityContextRepository(
                    httpSecurityRepository, new RequestAttributeSecurityContextRepository());
            http.setSharedObject(SecurityContextRepository.class, defaultRepository);
        }
    }
    else {
        this.sessionManagementSecurityContextRepository = securityContextRepository;
    }
    RequestCache requestCache = http.getSharedObject(RequestCache.class);
    if (requestCache == null) {
        if (stateless) {
            http.setSharedObject(RequestCache.class, new NullRequestCache());
        }
    }
    http.setSharedObject(SessionAuthenticationStrategy.class, getSessionAuthenticationStrategy(http));
    http.setSharedObject(InvalidSessionStrategy.class, getInvalidSessionStrategy());
}

```

stateless 是一个标记位，告诉 Spring Security 要不要依赖 HttpSession 来保存 SecurityContext，有四种策略：

```java

public enum SessionCreationPolicy {

    /**
     * Always create an {@link HttpSession}
     */
    ALWAYS,

    /**
     * Spring Security will never create an {@link HttpSession}, but will use the
     * {@link HttpSession} if it already exists
     */
    NEVER,

    /**
     * Spring Security will only create an {@link HttpSession} if required
     */
    IF_REQUIRED,

    /**
     * Spring Security will never create an {@link HttpSession} and it will never use it
     * to obtain the {@link SecurityContext}
     */
    STATELESS

}

```

默认是 IF_REQUIRED，有需要的话就创建 HttpSession，STATELESS 一般用在 JWT 体系，表示服务端无状态，不需要 HttpSession 存储 SecurityContext，因为 JWT 的逻辑就是根据 Bearer Token 每次进来都要进行解析，创建对应的 Authentication，服务端无需保存 Session 信息。

stateless 既不创建 HttpSession 来保存 Spring Security 的认证信息，也不从已有 HttpSession 中恢复认证信息，所不需要 SecurityContextRepository，可以从源码看到，如果是 stateless 模式，就会创建一个 NullSecurityContextRepository。

而在依赖 Session 的体系中，SessionManagementConfigurer 会准备下面的 SecurityContextRepository：

```java

DelegatingSecurityContextRepository defaultRepository = new DelegatingSecurityContextRepository(
                    httpSecurityRepository, new RequestAttributeSecurityContextRepository());

```

两个类的作用分别是：

- HttpSessionSecurityContextRepository: 让认证信息能够跨多个 HTTP 请求保存
- RequestAttributeSecurityContextRepository: 让认证信息能够在同一请求的 ERROR/ASYNC 等 dispatch 中恢复

然后把 Repository 放入 HttpSecurity 的共享对象中，交给 .securityContext() 使用。

SecurityContextHolderFilter 会从 SecurityContextRepository 中延迟获取 SecurityContext，放进 SecurityContextHolderStrategy 中。

---

## Session 模式和 Stateless 模式流程

Session 模式

```

请求携带 JSESSIONID
    ↓
SecurityContextRepository 从 HttpSession 恢复 SecurityContext
    ↓
放入 SecurityContextHolder（当前线程）
    ↓
Controller / Service / 权限判断
    ↓
登录成功、身份变化或退出时，保存/删除 Repository 中的 Context
    ↓
请求结束，清空 SecurityContextHolder

```

Stateless 模式

```

请求携带 JWT
    ↓
认证过滤器每次重新验证 JWT
    ↓
创建 Authentication 和 SecurityContext
    ↓
放入 SecurityContextHolder
    ↓
Controller / Service / 权限判断
    ↓
请求结束，清空 SecurityContextHolder

```

