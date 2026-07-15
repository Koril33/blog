---
title: "Spring Security配置SecurityFilterChain"
date: 2026-07-12T13:53:57
summary: "如何覆盖默认配置？"
---

## 目录

[TOC]

---

## 前言

Spring Security 的配置核心就是配置一系列的 SecurityFilterChain，之前的文章已经简单讨论过 Spring Boot 的自动装配机制，给引入 Spring Security 依赖后，提供了简单的默认配置。

本文简述如何自定义配置。

---

## WebSecurityConfigurerAdapter

在 Spring Security 老版本（5.x）之前，使用 WebSecurityConfigurerAdapter 来配置（需要一个配置类继承该类），但该类在新版本已经废弃，新版本使用配置类和 `@Bean` 来进行自定义配置。

参考[官方博客](https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter)

---

## WebSecurity 和 HttpSecurity

废弃掉了 WebSecurityConfigurerAdapter 后，官方推荐的配置类似：

```java
@Configuration
public class SecurityConfiguration {

    @Bean
    public WebSecurityCustomizer webSecurityCustomizer() {
        return (web) -> web.ignoring().antMatchers("/ignore1", "/ignore2");
    }


    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
        return http.build();
    }
}
```

简单来讲：

- WebSecurity：控制整个 Spring Security 的全局行为。
- HttpSecurity：控制某一条 Security Filter Chain 如何处理 HTTP 请求。
- WebSecurity 负责“是否进入安全体系”，HttpSecurity 负责“进入之后如何进行安全处理”。

如果在 WebSecurity 配置 ignoring，那么符合忽略规则的请求，会直接跳过 Spring Security 过滤器体系，直接进入 Servlet。

官方并不推荐滥用 WebSecurity 的 ignoring 来略过 security，更加推荐配置 HttpSecurity 的 `authorizeHttpRequests` 来精细控制需要不同权限的访问。

---

## HttpSecurity

很多默认配置都是 Spring Boot 通过 `@ConditionalOnMissingBean` 提供的，如果我们自己提供了自定的 Bean，那么相对应的默认配置 Bean 就会失效。

我们想要自定义 Spring Security 的行为，首先就是配置 SecurityFilterChain：

```java
@Configuration
public class CustomSecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        return http.build();
    }
}
```

我们依然使用了 Spring Boot 提供的默认的 HttpSecurity，但覆盖了 `defaultSecurityFilterChain`：

```java
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnDefaultWebSecurity
	static class SecurityFilterChainConfiguration {

		@Bean
		@Order(SecurityFilterProperties.BASIC_AUTH_ORDER)
		SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) {
			http.authorizeHttpRequests((requests) -> requests.anyRequest().authenticated());
			http.formLogin(withDefaults());
			http.httpBasic(withDefaults());
			return http.build();
		}

	}
```

此时我们没有做任何配置，我们可以开启 `formLogin`：

```java
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        http.formLogin(Customizer.withDefaults());
        return http.build();
    }
```

`Customizer.withDefaults()` 可以理解成：我启用这个功能，但不做任何额外配置，全部采用 Spring Security 的默认配置。

Customizer 是个函数式接口：

```java
@FunctionalInterface
public interface Customizer<T> {

	/**
	 * Performs the customizations on the input argument.
	 * @param t the input argument
	 */
	void customize(T t);

	/**
	 * Returns a {@link Customizer} that does not alter the input argument.
	 * @return a {@link Customizer} that does not alter the input argument.
	 */
	static <T> Customizer<T> withDefaults() {
		return (t) -> {
		};
	}

}
```

之所以这么写，是因为 HttpSecurity 的 `formLogin` 函数如下，接受的是 Customizer：

```java
	public HttpSecurity formLogin(Customizer<FormLoginConfigurer<HttpSecurity>> formLoginCustomizer) {
		formLoginCustomizer.customize(getOrApply(new FormLoginConfigurer<>()));
		return HttpSecurity.this;
	}
```

如果什么都不配置，就是传递一个空的 lambda 函数：

```java
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        http.formLogin(form -> {});
        return http.build();
    }
```

`Customizer.withDefaults()` 就是 form -> {}，只不过都使用前者的话，看起来更整齐统一。

那么如何知道对应的默认行为？答案在对应的 Configurer 中，比如 `formLogin` 就去 FormLoginConfigurer 以及它继承的父类 AbstractAuthenticationFilterConfigurer 中找：

```
HttpSecurity

↓

formLogin()

↓

FormLoginConfigurer

↓

AbstractAuthenticationFilterConfigurer

↓

init()

↓

configure()
```

所以想要知道可以配置哪些内容，查看对应的 Configurer，比如 `formLogin` 对应的 FormLoginConfigurer。

此时，我们仅仅配置了 `formLogin`，并没有显示启用任何保护，所以访问接口不会跳转到登陆页面。

但是相关的登录页都已准备好，请求 [http://localhost:8080/login](http://localhost:8080/login) 可以看到 Spring Security 提供的默认登录页。

### `securityMatcher`

HttpSecurity 的 `securityMatcher` 是用来决定请求是否会进入到当前配置的 SecurityFilterChain，如果不配置，这条 SecurityFilterChain 默认匹配所有请求，相当于：

```java
http.securityMatcher("/**")
```

大部分简单的应用只需要配置一条 SecurityFilterChain，所以不需要配置 `securityMatcher`，但是如果是复杂的应用，需要 FilterChainProxy 使用不同规则匹配应用不用的 SecurityFilterChain 的时候，就需要配置 `securityMatcher` 了。

例如，对于 /API/** 和除了 /API/** 之外，需要应用两个不同的 SecurityFilterChain，可以这么写：

```java
@Configuration
public class SecurityConfig {

    @Bean
    @Order(1)
    SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**") // 只处理 /api/** 的请求
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    @Order(2)
    SecurityFilterChain webSecurity(HttpSecurity http) throws Exception {
        http
            // 未写 securityMatcher：可匹配其余任意请求
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/css/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());

        return http.build();
    }
}
```

系统仅会调用第一个匹配的 SecurityFilterChain，比如 /API/message 就会匹配到 `apiSecurity` 这个 SecurityFilterChain，而 /other 无法匹配 /API/** 的规则，FilterChainProxy 会按照 `@Order` 的顺序继续往下找，这里就匹配到了 `webSecurity`。

关于多个 SecurityFilterChain 的解释可以参考[文档](https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-multi-securityfilterchain-figure)。

最佳时间是，如果有多个 SecurityFilterChain，那么 `@Order` 的顺序应该是从小范围的具体匹配到最后的最大范围的兜底。

因为 `@Order` 越小优先级越高，匹配到了第一个，就不会执行剩余的，所以越具体的匹配要放到最前面，最终一定要有一个兜底的匹配（就是不配置 `securityMatcher`），来匹配除了前面所有条件之外的剩余的请求：

```
/api/**
↓
/admin/**
↓
/actuator/**
↓
最后要有个兜底
/**
```

```java
@Bean
@Order(100) // 放在最后，兜底用
SecurityFilterChain defaultChain(HttpSecurity http) {

    http
        .authorizeHttpRequests(auth ->
            auth.anyRequest().authenticated());

    return http.build();
}
```

一旦使用了 `securityMatcher`，就一定要划分清楚每个 SecurityFilterChain 的匹配范围。

### `authorizeHttpRequests`

如果说 `securityMatcher` 是决定一个请求是否进入当前 SecurityFilterChain 的，那么 `authorizeHttpRequests` 就是决定进入了当前 Chain 以后，这个请求是否允许访问。

一个最简单的例子，我们在上面仅仅配置了 `formLogin` 的基础上，再配置 `authorizeHttpRequests`：

```java
@Configuration
public class WebSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
        http.formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```
重启应用，访问任意接口，发现会跳转到登录页，因为这一行的配置意思是：所有请求必须登录。

我们可以通过一些方式来配置哪些 URL 需要哪些权限才可以访问或者拒绝访问，HttpSecurity 的 `authorizeHttpRequests` 如下：

```java
	public HttpSecurity authorizeHttpRequests(
			Customizer<AuthorizeHttpRequestsConfigurer<HttpSecurity>.AuthorizationManagerRequestMatcherRegistry> authorizeHttpRequestsCustomizer) {
		ApplicationContext context = getContext();
		authorizeHttpRequestsCustomizer
			.customize(getOrApply(new AuthorizeHttpRequestsConfigurer<>(context)).getRegistry());
		return HttpSecurity.this;
	}
```

AuthorizationManagerRequestMatcherRegistry 继承自 AbstractRequestMatcherRegistry，AbstractRequestMatcherRegistry 中有个方法：

```java
	public C requestMatchers(String... patterns) {
		return requestMatchers(null, patterns);
	}
```

还有其他几个重载方法：

```java
requestMatchers(String... patterns) // 只按路径匹配
requestMatchers(HttpMethod method, String... patterns) // HTTP 方法 + 路径
requestMatchers(HttpMethod method) // 只按 HTTP 方法，不限制路径
requestMatchers(RequestMatcher... requestMatchers) // 你自己提供匹配器，规则可任意复杂
```

本是就是一个匹配器，匹配到的请求进行下一步判定——AuthorizeHttpRequestsConfigurer 中的各种权限规则方法：

- `permitAll()`：无条件允许，不要求认证
- `denyAll()`：无条件拒绝
- `hasRole("ADMIN")`：必须拥有 `ROLE_ADMIN`
- `hasAnyRole("ADMIN"`, "USER")：拥有 `ROLE_ADMIN` 或 `ROLE_USER` 中任意一个
- `hasAllRoles("ADMIN"`, "USER")：必须同时拥有 `ROLE_ADMIN` 和 `ROLE_USER`
- `hasAuthority("article:read")`：必须拥有精确权限 article:read
- `hasAnyAuthority("read"`, "write")：拥有 read 或 write 中任意一个
- `hasAllAuthorities("read"`, "write")：必须同时拥有 read 和 write
- authenticated()：只要已认证即可，包括 remember-me 认证
- `fullyAuthenticated()`：必须完整认证，不接受 remember-me 认证
- `rememberMe()`：只允许通过 remember-me 认证的用户
- anonymous()：只允许匿名用户，已登录用户不符合
- access()：支持自定义的 AuthorizationManager

一个简单的示例如下：

```java
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        http.authorizeHttpRequests(
                auth ->
                        auth
                                .requestMatchers("/open/**").permitAll()
                                .requestMatchers("/admin/**").hasRole("ADMIN")
                                .requestMatchers("/user/**").hasAnyRole("ADMIN", "USER")
                                .anyRequest().authenticated());
        http.formLogin(Customizer.withDefaults());
        return http.build();
    }
```

写法就是指定一个匹配器再指定匹配的权限判定规则。

上面的示例所作出的权限限制：

1. 匹配到 /open/** 的请求，全部放行，无需登录。
2. 匹配到 /admin/** 的请求，需要登陆且用户需要具有 `ROLE_ADMIN` 角色，才能访问。
3. 匹配到 /user/** 的请求，需要登录且用户需要具有 `ROLE_ADMIN` 或者 `ROLE_USER` 角色之一，才能访问。
4. 除以上匹配条件之外的请求，需要登录才能访问

---

## 用户管理

为了测试刚刚的 HttpSecurity 配置，这里简单提下 Spring Security 的用户管理部分。

Spring Boot 自动配置用户的配置类是：UserDetailsServiceAutoConfiguration：

```java
	@Bean
	InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties,
			ObjectProvider<PasswordEncoder> passwordEncoder) {
		SecurityProperties.User user = properties.getUser();
		List<String> roles = user.getRoles();
		return new InMemoryUserDetailsManager(User.withUsername(user.getName())
			.password(getOrDeducePassword(user, passwordEncoder.getIfAvailable()))
			.roles(StringUtils.toStringArray(roles))
			.build());
	}
```

默认使用的 UserDetailsManager 是 InMemoryUserDetailsManager 即将用户存储在内存中，可以在 SecurityProperties 中看到默认的用户名是 user，密码是一个随机生成的 UUID 值：

```java
	public static class User {

		/**
		 * Default user name.
		 */
		private String name = "user";

		/**
		 * Password for the default user name.
		 */
		private String password = UUID.randomUUID().toString();

		/**
		 * Granted roles for the default user name.
		 */
		private List<String> roles = new ArrayList<>();
	}
```

我们可以通过自己提供 UserDetailsSerivce 来覆盖 Spring 的默认配置，出于演示的目的，为了不引入额外的数据库依赖，依然使用 InMemoryUserDetailsManager，往里面加入三个自定义的用户：

```java
@Configuration
public class CustomUserConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }


    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder passwordEncoder) {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        UserDetails alice = User.withUsername("Alice").password(passwordEncoder.encode("123")).roles("ADMIN").build();
        UserDetails bob = User.withUsername("Bob").password(passwordEncoder.encode("123")).roles("USER").build();
        UserDetails cindy = User.withUsername("Cindy").password(passwordEncoder.encode("123")).build();
        manager.createUser(alice);
        manager.createUser(bob);
        manager.createUser(cindy);
        return manager;
    }
}
```

三个用户主要区别在于角色，Alice 具有 `ROLE_ADMIN` 角色，Bob 具有 `ROLE_USER` 角色，Cindy 没有任何角色。

对应的测试 controller：

```java
@RestController
public class HomeController {

    @GetMapping("/")
    public String home(Authentication authentication) {
        return "Hello World: " + authentication.getName() + "\n";
    }

    @GetMapping("/open")
    public String open(Authentication authentication) {
        String username = Objects.nonNull(authentication) ? authentication.getName() : "Anonymous";
        return "Open API: " + username + "\n";
    }

    @GetMapping("/admin")
    public String admin(Authentication authentication) {
        return "Hello Admin: " + authentication.getName() + "\n";
    }


    @GetMapping("/user")
    public String user(Authentication authentication) {
        return "Hello User: " + authentication.getName() + "\n";
    }
}
```

- / 匹配到的规则是 `.anyRequest().authenticated())`，只有登陆的用户才可以访问 /
- /open 匹配到的规则是 `.requestMatchers("/open/**").permitAll()`，登陆或者不登陆都可以访问 /open
- /admin 匹配到的规则是 `.requestMatchers("/admin/**").hasRole("ADMIN")`，只有拥有角色 `ROLE_ADMIN` 的用户能访问 /admin
- /user 匹配到的规则是 `.requestMatchers("/user/**").hasAnyRole("ADMIN", "USER")` 拥有角色 `ROLE_ADMIN` 或者 `ROLE_USER` 之一的角色能访问 /user

最终呈现的结果是

- Alice 能访问所有接口
- Bob 能访问 /, /open, /user 三个接口
- Cindy 能访问 /, /open 两个接口
- 未登录用户仅能访问 /open 一个接口

---

## 参考

1. [https://docs.spring.io/spring-security/reference/servlet/configuration/java.html#jc-httpsecurity](https://docs.spring.io/spring-security/reference/servlet/configuration/java.html#jc-httpsecurity)
2. [https://www.baeldung.com/spring-security-httpsecurity-vs-websecurity?utm_source=chatgpt.com](https://www.baeldung.com/spring-security-httpsecurity-vs-websecurity?utm_source=chatgpt.com)
