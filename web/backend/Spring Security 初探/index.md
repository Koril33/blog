---
title: "Spring Security 初探"
date: 2023-12-30T14:44:37+08:00
tags: []
featured_image: "images/background.jpg"
summary: "Spring Security 框架的基础概念和示例"
toc: true
---

## 安全性

在软件开发的发展历史中，软件安全性一直是一个关键性的因素，尽管作为软件系统的非功能性特性之一，但是我们不能简单的忽略它。

如果仅仅实现了功能性特性，例如银行系统的查看账户和转账功能，那么用户的隐私信息和其他安全有关的问题将得不到保障。遗憾的是，大部分系统从一开始，为了快速迭代或者单纯的偷懒，往往忽略对于安全性的代码投资，渐渐的，安全性技术负债越滚越多，到最后的某一个环节崩溃时，不单单是金钱的损失，更有可能带来信誉甚至生命的危险。

---

## Spring Security

Spring Security 是一个安全性框架，除了它之外，常见的安全性框架还有 Apache Shiro。没有框架的帮助，我们在常见的 Web 开发中，只能不断的一遍遍重写相似的验证代码，并且由于各个开发人员的水平参差不齐，对于安全性认证和授权的代码往往质量也不尽如人意，甚至可能直接忽略。

Spring Security 为我们带来了很多契约式设计，在固有接口的情况下，可以根据实际开发需求进行扩展与重写。

---

## 一些概念

在后续的一些讨论中，有些概念反复出现，在此进行明确。

### 验证

验证（Authentication），指的是软件系统通过某些方式，识别用户的过程。最常见的就是登录验证，用户通过提供用户名以及密码，来让系统判定当前用户的身份。常见的错误是，用户提供的凭证是错误的，用户信息根本不存在或者用户名对应的密码错误，此时系统将拒绝用户的进一步访问。

举个例子，某个政府大楼看守严密，在门口的保安人员手持一份可以出入大楼的名单，只有在名单上的人才可以进入大楼，这就是验证访问者的身份。

### 授权

一个软件的安全系统，仅仅有验证功能是可以的，但是还不够，再稍微复杂一些的情况下，我们需要对资源进行访问权限的划分，比如某些资源不是什么机密，可以对所有已登录用户进行开放，而有些资源比较重要，只希望某些管理人员才能看到，这时候就需要授权（Authorization）的功能。

举个例子，尽管你在保安人员的可以出入大楼的名单上，但这不意味着你可以进入大楼的每一扇门，保安会在验证通过后，交给你一张访问卡，这张卡只能带开大楼的部分门，你被授予的访问权限是被限制的。

### 凭证

指的是用户所持有的某种信息，这个信息能够帮助系统验证该用户的身份，以及授予相对应的权限，往往是指用户名和密码。

---

## 最简单的 Spring Security 应用

现在先不考虑使用额外的持久化存储（比如：MySQL，MongoDB）来存储用户信息，凭证和权限。仅仅引入 Spring Web 和 Spring Security 来看下最简单的应用是如何保护 Web API 的。

pom.xml 需要引入依赖：

```
<!-- spring security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- spring web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

然后创建一个最简单 Controller：

```java
@RestController
@RequestMapping("/api")
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello, spring security!";
    }
}
```

启动后，看到打印的日志和平时未引入 Spring Security 有所不同：

![](./images/1.jpg)

多了个临时的密码，这个密码就是之后要访问后台任何接口的凭证。

使用 curl 请求刚刚的接口，如果不进行登录的话，是无法访问的：

![](./images/2.jpg)

并没有打印任何内容，curl 有个参数 -i 可以打印响应头的信息：

![](./images/3.jpg)

通过 response header 可以清晰地看到返回了 401，表示 Unauthorized，即未经授权，缺少所请求资源的有效身份验证凭据。

但是我个人觉得这里的英文名称稍有歧义，因为我们没有输入凭证信息进行验证，和授权并无关系。

除了 401 之外，还有个叫 403 的 HTTP 状态码，表示 Forbidden，即服务器理解该请求但拒绝授权。有的人认为 401 其实是 authentication error，而 403 才是 authorization error，具体的争辩可以参考以下问答：

https://stackoverflow.com/questions/3297048/403-forbidden-vs-401-unauthorized-http-responses

401 和 403 的定义可以参考 mdn 的标准文档：

* https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/401
* https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/403

如果想要访问到刚才创建的接口，那么我们必须提供正确的用户凭证：

```
用户名: user
密码: 刚刚控制台打印出来的 UUID
```

curl 有个 -u 的选项可以设置 http basic 认证的用户名和密码：

![](./images/4.jpg)

可以看到输入了正确的用户名和密码之后，收到了应用的响应。

那么，curl 在这个过程中做了些什么事呢？具体的来问，curl 是如何通过 http 协议，将 username 和 password 发给服务器进行验证的？

为了弄清楚这一点，curl 有个 -v 的选项可以把完整的请求过程打印出来：

![](./images/5.jpg)

curl 实际发送的是一个 Authorization 的 http header，里面包含了 username 和 password 经过特殊格式编码后的字符串，具体的规则是：使用英文冒号将 username 和 password 隔开，然后对其进行 Base64 编码，最后加上 “Basic ” 的前缀，放入到 Authorization 的 header 中：

```
Authrorization: Basic Base64.encode("{username}:{password}")
```

可以借助 https://www.base64encode.org/ 来看看编码后的结果：

![](./images/6.jpg)

和 curl 发送的一致，也就是说，下面两种请求，都是可以通过 http 身份验证的：

```
# curl 会自动进行 Base64 编码
curl.exe -u "user:04015237-e610-4fa8-a174-a0fcb68c4fce" http://localhost:8080/api/hello

# 也可以手动设置 Authrorization 的 header 值
curl.exe -H "Authorization: Basic dXNlcjowNDAxNTIzNy1lNjEwLTRmYTgtYTE3NC1hMGZjYjY4YzRmY2U=" http://localhost:8080/api/hello
```

完整的 Http authentication 如图所示：

![](./images/7.jpg)

---

## Spring Security 基本认证流程

我们仅仅是引入了一个依赖，没有写任何的，有关于安全，验证的代码，框架已经帮我们做了很多的事情：

1. 保护所有 API，禁止未通过验证的用户访问
2. 生成默认的一个用户，名称为 user
3. 每次启动应用，都会随机生成 user 对应的 password
4. 默认使用了 Http Authentication 的验证方式

和 Spring Boot 类似，Spring Security 默认就有一套配置，当然这种配置比较简单，仅仅用作演示和查看依赖是否被正确的添加，并不能在实际的生产环境中使用。

接下来，简述下 Spring Security 基本的认证流程。

### 过滤器链

Spring Security 的核心是过滤器链，在正常的 Servlet 应用程序中，我们常常使用 Servlet Filter 来对请求做出一些处理，比如认证，日志记录，修改请求内容等等，多个过滤器可以组成过滤器链，置于 Servlet 之前，关于 Servlet Filter 可以参考我之前的文章：
https://korilweb.cn/backend/%E6%8B%A6%E6%88%AA%E5%99%A8%E5%92%8C%E8%BF%87%E6%BB%A4%E5%99%A8/

![](./images/8.jpg)

Spring Security 的本质也是基于 Servlet Filter 的（本文仅讨论 Servlet 下的 Spring Security，WebFlux 它也支持），

### DelegatingFilterProxy 和 FilterChainProxy

Spring 中有一个类叫 DelegatingFilterProxy，它是一个 Servlet Filter 的代理类，用这个类的好处是通过 Spring 容器来管理 Servlet Filter 的生命周期，Spring Security 的过滤器链——FilterChainProxy 就是由它来代理的。

![](./images/9.jpg)

FilterChainProxy 包含了一组 Security Filter，用于进行各种安全性验证。

### AuthenticationFilter 和 AuthenticationManger

对于身份的验证，就属于其中一个过滤器——AuthenticationFilter，AuthenticationFilter 将对身份验证的职责委派给 AuthenticationManager，它是认证的主要策略接口：

```java
public interface AuthenticationManager {
	Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

如果验证通过，AuthenticationManger 会返回一个 Authentication 对象，如果认证失败则会抛出 AuthenticationException 异常，如果无法决定则返回 null。

### AuthenticationProvider 和 ProviderManger

AuthenticationManger 最常见的实现类是 ProviderManger，ProviderManger 将认证逻辑委托给一系列的 AuthenticationProvider：

```java
public interface AuthenticationProvider {
    
	Authentication authenticate(Authentication authentication) throws AuthenticationException;

	boolean supports(Class<?> authentication);
}
```

AuthenticationProvider 有点像 AuthenticationManger，不过它多了一个 supports 方法，来判定当前的 AuthenticationProvider 是否支持认证。换句话说，AuthenticationProvider 组成了一个链，由 ProviderManger 来代理，认证一个用户可能不仅仅只有 username + password 的方式，也可能通过别的方式来认证，所以 AuthenticationProvider 链就很有必要了。

### UserDetailsManger 和 PasswordEncoder

假设使用的是 username + password 的方式，就需要有获取 User 详情的抽象，以及关于密码加密和匹配的抽象，AuthenticationProvider 将这部分的责任代理给了 UserDetailsManger 和 PasswordEncoder。

UserDetailsManger 和 PasswordManager 是相伴而生的，当我们自定义了 UserDetailsManger 的实现类之后，我们还必须指定一个 PasswordEncoder。

下面是一个框架图：

![](./images/10.jpg)

### 默认配置

到目前为止，我们没有编写任何安全相关的代码，所有涉及的类均使用 Spring Security 提供的默认配置。

username 和 password 的默认配置存放在 SecurityProperties 中的 User 类当中：

```java
@ConfigurationProperties(prefix = "spring.security")
public class SecurityProperties {
	// 省略部分代码
	public static class User {
		private String name = "user";
		private String password = UUID.randomUUID().toString();
	}
}
```

SecurityProperties 头上的注解是 ConfigurationProperties，说明我们可以在 application.properties 中修改 默认的用户名和密码：

```properties
spring.security.user.name=koril
spring.security.user.password=123456
```

重新启动 SpringBoot 程序，发现原来的那个随机的 UUID 密码提示消失了，我们现在需要使用配置文件的 username 和 password 通过验证：

```
curl.exe -u "koril:123456" http://localhost:8080/api/hello
```

UserDetailsService 的默认配置放在 UserDetailsServiceAutoConfiguration 当中：

```java
@AutoConfiguration
@ConditionalOnClass(AuthenticationManager.class)
@ConditionalOnBean(ObjectPostProcessor.class)
@ConditionalOnMissingBean(
		value = { AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class,
				AuthenticationManagerResolver.class },
		type = { "org.springframework.security.oauth2.jwt.JwtDecoder",
				"org.springframework.security.oauth2.server.resource.introspection.OpaqueTokenIntrospector",
				"org.springframework.security.oauth2.client.registration.ClientRegistrationRepository",
				"org.springframework.security.saml2.provider.service.registration.RelyingPartyRegistrationRepository" })
public class UserDetailsServiceAutoConfiguration {

	private static final String NOOP_PASSWORD_PREFIX = "{noop}";

	private static final Pattern PASSWORD_ALGORITHM_PATTERN = Pattern.compile("^\\{.+}.*$");

	private static final Log logger = LogFactory.getLog(UserDetailsServiceAutoConfiguration.class);

	@Bean
	@Lazy
	public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties,
			ObjectProvider<PasswordEncoder> passwordEncoder) {
         // 从 security properties 当中获取用户
		SecurityProperties.User user = properties.getUser();
		List<String> roles = user.getRoles();
         // 构建一个 InMemoryUserDetailsService
		return new InMemoryUserDetailsManager(User.withUsername(user.getName())
			.password(getOrDeducePassword(user, passwordEncoder.getIfAvailable()))
			.roles(StringUtils.toStringArray(roles))
			.build());
	}
}
```

当我们没有添加自定义的 AuthenticationManager，AuthenticationProvider.class，UserDetailsService，AuthenticationManagerResolver 的时候，Spring Security 会自动为我们配置一个将用户信息存放在内存当中的 InMemoryUserDetailsManager。

默认下，Spring Security 使用 Http Basic 认证方式，并且对所有接口进行保护，这部分的配置在 SpringBootWebSecurityConfiguration 中：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
class SpringBootWebSecurityConfiguration {

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnDefaultWebSecurity
	static class SecurityFilterChainConfiguration {

		@Bean
		@Order(SecurityProperties.BASIC_AUTH_ORDER)
		SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
             // 对所有请求都进行认证
			http.authorizeRequests().anyRequest().authenticated();
             // 表单登录
			http.formLogin();
             // Http Basic 认证
			http.httpBasic();
			return http.build();
		}
	}
}
```

如果用户自定义了 WebSecurityConfigurerAdapter 或者 SecurityFilterChain 的 Bean，那么这个默认配置就不会启用了。

---

## 重写默认配置

### 重写用户管理配置

本节演示如何重写默认的配置项，以 UserDetailsService 为例，但是我们不自定义实现，而是使用之前提到的 InMemoryUserDetailsManager。

新建一个 Spring 的配置类，并提供 UserDetailsService 的 Bean：

```java
@Configuration
public class ProjectConfig {

    @Bean
    public UserDetailsService userDetailsService() {
        return new InMemoryUserDetailsManager();
    }
}
```

@Configuration 将此类标记为配置类，@Bean 则指示 Spring 添加我们自定义的 UserDetailsService 的实现类到 Spring 上下文当中。

此时，我们的配置替代了 Spring Security 的默认实现，就像上一小节提到的，Spring Security 对于 UserDetailsService 的默认配置也是使用 InMemoryUserDetailsManager，但它会从 SecurityProperties 当中读取用户名和密码，而我们的配置类并没有指定用户信息，所以目前为止，我们无法访问任何接口。

指定用户的信息，需要提供用户名和密码：

```java
@Configuration
public class ProjectConfig {

    @Bean
    public UserDetailsService userDetailsService() {

        InMemoryUserDetailsManager userDetailsManager = new InMemoryUserDetailsManager();
        // 创建一个用户
        UserDetails user = User.withUsername("alice").password("123456").build();
        userDetailsManager.createUser(user);

        return userDetailsManager;
    }
}
```

但仅仅提供这两个信息还不够，启动应用后会爆出以下错误：

![](./images/11.jpg)

原因是，构造 User 的时候，必须指定一个权限信息，尽管目前没什么作用：

```java
@Configuration
public class ProjectConfig {

    @Bean
    public UserDetailsService userDetailsService() {

        InMemoryUserDetailsManager userDetailsManager = new InMemoryUserDetailsManager();
        // 创建一个用户
        UserDetails user = User.withUsername("alice").password("123456").authorities("read").build();
        userDetailsManager.createUser(user);

        return userDetailsManager;
    }
}
```

现在虽然可以正常访问应用了，但尽管输入了正确的用户名和密码，但是还是返回 401 信息，而且控制台爆出了以下错误：

![](./images/12.jpg)

之前提到了，PasswordEncoder 和 UserDetailsService 是相伴相生的，我们自定义了 UserDetailsService 之后，同时也必须手动指定一个 PasswordEncoder：

```java
@Configuration
public class ProjectConfig {

    @Bean
    public UserDetailsService userDetailsService() {

        InMemoryUserDetailsManager userDetailsManager = new InMemoryUserDetailsManager();
        // 创建一个用户
        UserDetails user = User.withUsername("alice").password("123456").authorities("read").build();
        userDetailsManager.createUser(user);

        return userDetailsManager;
    }

    // 指定一个 PasswordEncoder
    @Bean
    public PasswordEncoder passwordEncoder() {
        // 绝不建议在生产环境使用 NoOpPasswordEncoder，因为它不会对密码进行加密，仅仅作为普通文本对待
        return NoOpPasswordEncoder.getInstance();
    }
}
```

终于，可以正常访问了。

### 重写接口授权配置

Spring Security 默认使用 Http Basic 认证方式，但它安全性低，并不适用大部分的程序架构。想要重写授权配置，可以扩展 WebSecurityConfigurerAdapter 类，然后重写 configure(HttpSecurity http) 方法：

```java
@Configuration
public class ProjectConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ...一些配置代码
    }
}
```

在大部分项目中依然能看到这样的配置方式，但是仔细阅读它的 javadoc 就会发现，该类已经 Deprecated，最新的方式是使用注入 SecurityFilterChain 的 Bean 来实现自定义配置，也就是我们在上节中看到的 SecurityFilterChainConfiguration 的配置方式：

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    // ...一些配置代码
    return http.build();
}
```

之后的代码都会采用 Bean 的方式来配置。

默认的配置是，所有接口都需要认证，如果我们希望放行所有接口，非常简单，将 .anyRequest().authenticated() 改成 .anyRequest().permitAll() 即可：

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    // 不进行验证，统统放行
    http.authorizeRequests().anyRequest().permitAll();
    return http.build();
}
```

使用 curl 验证，发现不带用户凭证也可以访问接口了：

```
curl.exe http://localhost:8080/api/hello
// 返回 Hello, spring security!
```

### 重写身份验证类

AuthenticationProvider 将身份验证时，需要的用户查询和密码校验委托给了 UserDetailsService 和 PasswordEncoder，我们也可以重写 AuthenticationProvider 的逻辑，下面的代码，使用自定义的实现，取代了本该交给 UserDetailsService 和 PasswordEncoder 的任务：

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        String username = authentication.getName();
        String password = (String) authentication.getCredentials();

        // 自定义身份验证的逻辑
        if ("bob".equals(username) && "123456".equals(password)) {
            return new UsernamePasswordAuthenticationToken(username, password, new HashSet<>());
        }

        throw new AuthenticationCredentialsNotFoundException("authentication error!");
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

上面的代码仅仅是展示如何自定义 AuthenticationProvider 的方式，用 if-else 子句实现了用户查找和密码验证的功能，但在实际开发中，还是遵循最佳实践，将用户查找和密码验证委托给 UserDetailsService 和 PasswordEncoder。

### 职责分离

上面的例子为了方便，将所有的 @Bean 和配置放在了同一个配置类——ProjectConfig。而在最佳实践中，需要将配置文件进行拆分，比如，将用户管理相关的放在一个配置文件中，而将授权管理相关的放在另一个配置文件中。

定义用户和密码相关的配置类：

```java
@Configuration
public class UserManagementConfig {


    @Bean
    public UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager userDetailsManager = new InMemoryUserDetailsManager();
        // 创建一个用户
        UserDetails user = User.withUsername("alice").password("123456").authorities("read").build();
        userDetailsManager.createUser(user);

        return userDetailsManager;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }
}
```

定义用于授权管理的配置类：

```java
@Configuration
public class WebAuthenticationConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.httpBasic();
        http.authorizeRequests().anyRequest().authenticated();
        return http.build();
    }

}
```

---

## 管理用户

本节将重点放在用户相关的接口和实现上面，之前简单提到了 UserDetailsSerivce，它主要用于查询用户信息，将查到的信息交给 AuthenticationProvider，配合 PasswordEncoder 来验证用户身份。它在整个框架中，作为获取用户信息的 DAO 接口。

```java
public interface UserDetailsService {
    
    // UserDetailsService 仅仅有一个方法，接收一个用户名，返回一个完整的 user record，返回值永远不会是 null
    // 如果无法根据 username 找到对应的用户信息，那么就会抛出 UsernameNotFoundException 的异常
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

还有一个接口叫 UserDetailsManager，UserDetailsManager 继承了 UserDetailsService，当接口继承了接口，我们可以知道，它从抽象变得具体，并且会额外有些功能：

```java
public interface UserDetailsManager extends UserDetailsService {

	/**
	 * 创建一个 user
	 */
	void createUser(UserDetails user);

	/**
	 * 更新一个 user
	 */
	void updateUser(UserDetails user);

	/**
	 * 删除一个 
	 */
	void deleteUser(String username);

	/**
	 * 更改当前用户的密码
	 */
	void changePassword(String oldPassword, String newPassword);

	/**
	 * 检查用户是否存在于系统中
	 */
	boolean userExists(String username);

}
```

UserDetailsService 仅仅实现了 DAO 的查找功能，但是 UserDetailsManager 在它的基础之上，提供了更多的方法——增删改。

这种契约的分离，很好地体现了接口的分离原则，我们如果在系统中，仅仅需要校验用户是否存在，那么使用 UserDetailsService 接口就足够了，如果还需要管理用户，那么可以选择 UserDetailsManager。分离接口，提供了更大的灵活性，我们不需要在不必要的时候，实现多余的方法。

UserDetailsService 和 UserDetailsManager 对于 User 这个 entity 的抽象，使用了 UserDetails 接口来表示：

```java
public interface UserDetails extends Serializable {

	/**
	 * 返回授予用户的权限，返回值不会为 null
	 */
	Collection<? extends GrantedAuthority> getAuthorities();

	/**
	 * 返回用于验证用户身份的密码
	 */
	String getPassword();

	/**
	 * 返回用于验证用户身份的用户名
	 */
	String getUsername();

	/**
	 * 表示账号是否过期，过期的账号无法通过验证
	 * true: 没过期，正常通过验证
	 * false: 过期账号，无法通过验证
	 */
	boolean isAccountNonExpired();

	/**
	 * 表示账号是否被锁定，锁定的账号无法通过验证
	 * true: 未锁定，正常通过验证
	 * false: 被锁定，无法通过验证
	 */
	boolean isAccountNonLocked();

	/**
	 * 表示凭据是否过期，过期的凭据无法通过验证
	 * true: 凭据未过期，正常通过验证
	 * false: 凭据已过期，无法通过验证
	 */
	boolean isCredentialsNonExpired();

	/**
	 * 表示账号是否启用，非启用账号无法通过验证
	 * true: 启用，正常通过验证
	 * false: 未启用，无法通过验证
	 */
	boolean isEnabled();

}
```

UserDetails 是 Spring Security 框架和开发者之间的契约，约定了框架是如何理解并描述用户信息的方式。

后续内容将围绕这几个类，进行展开：

* UserDetails
* GrantedAuthority
* UserDetailsService
* UserDetailsManager

下面是它们之间的关系图：

![](./images/13.jpg)

### UserDetails

UserDetails 是 Spring Security 框架所理解的用户，所包含的 7 种方法中的两种，getUsername 和 getPassword 用来做身份验证，另外 5 种与授权用户访问资源有关。getAuthorities 用来返回授予用户的权限组，所谓权限就是指细粒度的权力或者角色，而角色是对一系列的权限进行分组。

另外四种，可以使账户拥有以下约束条件：

1. 使账户过期
2. 使凭据过期
3. 锁定账号
4. 禁用账号

并不是所有系统都需要这么细致的约束，如果不需要使用，return true 即可。

### GrantedAuthority

权限表示一个用户在系统中，能够做什么，能够访问哪些资源，Spring Security 为权限作出的抽象就是 GrantedAuthority 对象：

```java
public interface GrantedAuthority extends Serializable {

	/**
	 * 获取权限的名称
	 */
	String getAuthority();

}
```

要创建一个权限，只需要为权限找到一个合适的名称即可，之后在编写授权规则的时候可以引用这个名称。

可以使用 lambda 表达式创建一个权限对象，也可以使用框架提供的一个实现——SimpleGrantedAuthority 来创建一个不可变实例的权限对象：

```java
GrantedAuthority authority1 = () -> "READ";
GrantedAuthority authority2 = new SimpleGrantedAuthority("WRITE");
```

