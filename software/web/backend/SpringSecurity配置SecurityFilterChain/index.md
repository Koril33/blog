---
title: "SpringSecurity配置SecurityFilterChain"
date: 2026-07-12T13:53:57
summary: "如何覆盖默认配置？"
---

## 前言

Spring Security 的配置核心就是配置一系列的 SecurityFilterChain,之前的文章已经简单讨论过 SpringBoot 的自动装配机制，给引入 Spring Security 依赖后，提供了简单的默认配置。

本文简述如何自定义配置。

---

## WebSecurityConfigurerAdapter

在 Spring Security 老版本（5.x）之前，使用 WebSecurityConfigurerAdapter 来配置（需要一个配置类继承该类），但该类在新版本已经废弃，新版本使用配置类和@Bean来进行自定义配置。

参考[官方博客](https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter)

---

## WebSecurity和HttpSecurity

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

如果在 WebSecurity 配置 ignoring,那么符合忽略规则的请求，会直接跳过 Spring Security 过滤器体系，直接进入 Servlet。

官方并不推荐滥用 WebSecurity 的 ignoring 来略过 security,更加推荐配置 HttpSecurity 的 authorizeHttpRequests 来精细控制需要不同权限的访问。

---

## HttpSecurity

很多默认配置都是 SpringBoot 通过@ConditionalOnMissingBean提供的，如果我们自己提供了自定的 Bean,那么相对应的默认配置 Bean 就会失效。

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

我们依然使用了 Spring Boot 提供的默认的 HttpSecurity，但覆盖了 defaultSecurityFilterChain：

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

此时我们没有做任何配置，我们可以开启 formLogin：

```java

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        http.formLogin(Customizer.withDefaults());
        return http.build();
    }

```

Customizer.withDefaults() 可以理解成：我启用这个功能，但不做任何额外配置，全部采用 Spring Security 的默认配置。

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

之所以这么写，是因为 HttpSecurity 的 formLogin 函数如下，接受的是 Customizer：

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

Customizer.withDefaults() 就是 form -> {}，只不过都使用前者的话，看起来更整齐统一。

那么如何知道对应的默认行为？答案在对应的 Configurer 中，比如 formLogin 就去 FormLoginConfigurer 以及它继承的父类 AbstractAuthenticationFilterConfigurer 中找：

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

---

## 参考

1. https://docs.spring.io/spring-security/reference/servlet/configuration/java.html#jc-httpsecurity
2. https://www.baeldung.com/spring-security-httpsecurity-vs-websecurity?utm_source=chatgpt.com
