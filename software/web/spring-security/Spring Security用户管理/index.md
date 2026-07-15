---
title: "Spring Security用户管理"
date: 2026-07-14T10:32:10
summary: "在 Spring Security 框架下如何表示用户，如何管理用户？"
---

## 目录

[TOC]

---

## 前言

对于框架而言需要和框架的使用者约定好组件的抽象，这样双方操作的是接口，保证了解耦和可扩展性。

Spring Security 认证体系中一个重要的抽象就是用户，以及用户管理。

本文将讲述 Spring Security 框架中用户相关的接口和自定义实现。

---

## 用户

用户就是使用系统的人（或者机器），通常都有用户名，密码，邮箱，手机号，地址，是否启用，最近登录时间，创建时间等等信息。但在安全框架的层面，一些信息是不必要的（比如：地址，生日），Spring Security 只关心以下一些内容：

- 用户名
- 加密后的密码
- 权限或角色
- 账号是否过期？
- 账号是否锁定？
- 密码是否过期？
- 账号是否启用？

针对以上的内容，框架抽象出了一个用户的接口来表示框架所关心的，与安全和认证流程相关的用户核心信息：

```java
public interface UserDetails extends Serializable {

	/**
	 * Returns the authorities granted to the user. Cannot return <code>null</code>.
	 * @return the authorities, sorted by natural key (never <code>null</code>)
	 */
	Collection<? extends GrantedAuthority> getAuthorities();

	/**
	 * Returns the password used to authenticate the user. Can be null if the user has not
	 * specified a password (e.g. the user Passkeys instead).
	 * @return the password
	 */
	@Nullable String getPassword();

	/**
	 * Returns the username used to authenticate the user. Cannot return
	 * <code>null</code>.
	 * @return the username (never <code>null</code>)
	 */
	String getUsername();

	/**
	 * Indicates whether the user's account has expired. An expired account cannot be
	 * authenticated.
	 * @return <code>true</code> if the user's account is valid (ie non-expired),
	 * <code>false</code> if no longer valid (ie expired)
	 */
	default boolean isAccountNonExpired() {
		return true;
	}

	/**
	 * Indicates whether the user is locked or unlocked. A locked user cannot be
	 * authenticated.
	 * @return <code>true</code> if the user is not locked, <code>false</code> otherwise
	 */
	default boolean isAccountNonLocked() {
		return true;
	}

	/**
	 * Indicates whether the user's credentials (password) has expired. Expired
	 * credentials prevent authentication.
	 * @return <code>true</code> if the user's credentials are valid (ie non-expired),
	 * <code>false</code> if no longer valid (ie expired)
	 */
	default boolean isCredentialsNonExpired() {
		return true;
	}

	/**
	 * Indicates whether the user is enabled or disabled. A disabled user cannot be
	 * authenticated.
	 * @return <code>true</code> if the user is enabled, <code>false</code> otherwise
	 */
	default boolean isEnabled() {
		return true;
	}

}
```

这个接口定义了 Spring Security 框架眼中的用户信息，框架的运行逻辑就依赖于这个抽象。

框架提供了一个实现：`org.springframework.security.core.userdetails.User`，这个实现为我们日后自己去实现一个 UserDetails 提供了很好的参考。

### CredentialsContainer

首先是继承的接口，除了 UserDetails 之外，还继承了 CredentialsContainer：

```java
public interface CredentialsContainer {

	void eraseCredentials();

}
```

这个接口的目的就是为了清除凭证这个敏感信息，也就是密码，User 的 Override 如下：

```java
	@Override
	public void eraseCredentials() {
		this.password = null;
	}
```

在 ProviderManager 调用完认证 Provider 后，会去调用 `eraseCredentials`，确保敏感信息完成认证后就擦除，而不是一直保存：

```java
if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
	// Authentication is complete. Remove credentials and other secret data
	// from authentication
	((CredentialsContainer) result).eraseCredentials();
}
```

### builder

User 还提供了 builder 模式，可以链式调用，构造一个完整的 User：

```java
	/**
	 * Creates a UserBuilder with a specified username
	 * @param username the username to use
	 * @return the UserBuilder
	 */
	public static UserBuilder withUsername(String username) {
		return builder().username(username);
	}

	/**
	 * Creates a UserBuilder
	 * @return the UserBuilder
	 */
	public static UserBuilder builder() {
		return new UserBuilder();
	}
```

### equals,hashcode,`toString`

从 equals 和 hashcode 可以看到 User 的实现，是用 username 作为唯一性判定值的：

```java
	@Override
	public boolean equals(@Nullable Object obj) {
		if (obj instanceof User user) {
			return this.username.equals(user.getUsername());
		}
		return false;
	}

	/**
	 * Returns the hashcode of the {@code username}.
	 */
	@Override
	public int hashCode() {
		return this.username.hashCode();
	}
```

`toString` 故意隐藏了密码，这也很关键，不要将关键信息意外泄露在打印的日志中：

```java
	@Override
	public String toString() {
		StringBuilder sb = new StringBuilder();
		sb.append(getClass().getName()).append(" [");
		sb.append("Username=").append(this.username).append(", ");
		sb.append("Password=[PROTECTED], ");
		sb.append("Enabled=").append(this.enabled).append(", ");
		sb.append("AccountNonExpired=").append(this.accountNonExpired).append(", ");
		sb.append("CredentialsNonExpired=").append(this.credentialsNonExpired).append(", ");
		sb.append("AccountNonLocked=").append(this.accountNonLocked).append(", ");
		sb.append("Granted Authorities=").append(this.authorities).append("]");
		return sb.toString();
	}
```

### 自定义实现

User 的实现给了我们很多的参考，但是在实际项目中，用户表可能会存在更多的字段，一个典型的用户类可能如下：

```java
@Entity
public class AppUser {
    
    private String username;
    private String password;
    private String firstName;
    private String lastName;
    private String email;
    private String phoneNumber;
    private String address;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // 省略 getter setter equals hashcode toString 方法
}
```

这里用的是 JPA 的注解，表示 AppUser 是关系型数据库用户表的映射，问题在于，如果我们简单的让 AppUser 去 implement UserDetails 接口，那么整个 AppUser 类就会变得非常臃肿，职责也不明确：

```java
@Entity
public class AppUser implements UserDetails {

    private String username;
    private String password;
    private String firstName;
    private String lastName;
    private String email;
    private String phoneNumber;
    private String address;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of();
    }

    @Override
    public @Nullable String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return UserDetails.super.isAccountNonExpired();
    }

    @Override
    public boolean isAccountNonLocked() {
        return UserDetails.super.isAccountNonLocked();
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return UserDetails.super.isCredentialsNonExpired();
    }

    @Override
    public boolean isEnabled() {
        return UserDetails.super.isEnabled();
    }
}
```

AppUser 除了承担数据库实体类的职责，还需要将表的部分字段信息转换成 Spring Security 需要的信息，来构建 Authentication。

所以，为了明确的划分职责，最佳实践是，提供两个类，一个是 AppUser 数据库实体类，用来承载应用实际的用户信息，另一个是 SecurityUser，用来桥接 AppUser 和 Spring Security 所需要的 UserDetails：

```java
@Entity
public class AppUser {
    
    private String username;
    private String password;
    private String firstName;
    private String lastName;
    private String email;
    private String phoneNumber;
    private String address;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // 省略 getter setter equals hashcode toString 方法
}


public class SecurityUser implements UserDetails {

    private final AppUser appUser;

    public SecurityUser(AppUser appUser) {
        this.appUser = appUser;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of();
    }

    @Override
    public @Nullable String getPassword() {
        return this.appUser.getPassword();
    }

    @Override
    public String getUsername() {
        return this.appUser.getUsername();
    }
}
```

划分两个类，这样职责明确：AppUser 就是表示业务用户实体信息，而 SecurityUser 则用来走 Spring Security 的认证授权等流程。

在一些小型系统中，我们也可以不自定义 SecurityUser，直接使用 Spring Security 提供的 User，在 UserDetailsService 的 `loadUserByUsername` 实现方法中进行转换：

```java
@Override
public UserDetails loadUserByUsername(String username) {
    AppUser appUser = userRepository.findByUsername(username)
            .orElseThrow(() ->
                    new UsernameNotFoundException(username));

    return User.withUsername(appUser.getUsername())
            .password(appUser.getPassword())
            .roles("USER")
            .build();
}
```

总体思路就是：

```
数据库业务层面的 AppUser

       ↓ 转换

Spring Security UserDetails

       ↓ 进入 Spring Security 流程

Authentication.principal
```

---

## 权限和角色

大部分安全系统都有权限和角色的概念，在 RBAC 模型中，角色可以理解成权限的集合。

Spring Security 底层只有权限（Authority），角色（Role）是一种特殊的权限，带有前缀 ROLE_。

框架对于权限的抽象提供了 GrantedAuthority：

```java
public interface GrantedAuthority extends Serializable {
	@Nullable String getAuthority();
}
```

可以从 `getAuthority` 返回值看到，无论是权限还是角色，底层本质就是字符串：

```
ROLE_USER
ROLE_ADMIN
user:read
user:write
```

本质是字符串，但二者在业务层面存在差别，角色一般用来表示用户的身份或者岗位，权限则表示用户可以做什么，可以访问，可以修改什么内容。

一个用户可以拥有数个角色和权限，这在 RBAC 里面是标准行为：

```
// Alice 可以同时是管理员和审计师

  Alice
    |
	ROLE_ADMIN
	├── user:read
	├── user:create
	├── user:update
	├── user:delete
	└── report:export
    |
	ROLE_AUDITOR
	├── user:read
	└── report:read
```

### roles 和 authorities

回头来看 User 的 roles 和 authorities，它们的目的相同，都是为了填充 User 的 authorities：

```java
public static final class UserBuilder {
	private List<GrantedAuthority> authorities = new ArrayList<>();


	public UserBuilder roles(String... roles) {
		List<GrantedAuthority> authorities = new ArrayList<>(roles.length);
		for (String role : roles) {
			Assert.isTrue(!role.startsWith("ROLE_"),
					() -> role + " cannot start with ROLE_ (it is automatically added)");
			authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
		}
		return authorities(authorities);
	}


	public UserBuilder authorities(Collection<? extends GrantedAuthority> authorities) {
		Assert.notNull(authorities, "authorities cannot be null");
		this.authorities = new ArrayList<>(authorities);
		return this;
	}
}
```

区别在于 roles 会给入参挨个添加 ROLE_ 这个前缀，以下写法是等价的：

```java
.roles("ADMIN", "USER")

.authorities("ROLE_ADMIN", "ROLE_USER")
```

需要注意，多次调用 roles() 或 authorities() 不是可靠的累加方式，因为后一次通常会替换前一次：

```java
User.withUsername("Alice")
        .roles("ADMIN")
        .authorities("user:read")
        .build();
```

alice 最终只有 user:read 权限，`ROLE_ADMIN` 被覆盖了。

简单的理解是 roles() 基于 authorities()，默认会添加 ROLE_ 前缀，本质都是字符串，除此之外没有太多特殊之处。

### 创建 GrantedAuthority

GrantedAuthority 是个接口，并且仅包含一个函数，所以可以用 lambda 表达式创建，或者使用 Spring Security 提供的一个基本的实现类 SimpleGrantedAuthority 来创建，以创建一个 user:read 的权限为例：

```java
GrantedAuthority first = () -> "user:read";
GrantedAuthority second = new SimpleGrantedAuthority("user:read");
```

---

## 用户管理

到目前位置，已经理清了 Spring Security 对于用户和权限抽象的两个接口：

- UserDetails
- GrantedAuthority

下一步就是看看 Spring Security 对于用户管理（对于实体信息的增删改查）是如何抽象的。

### UserDetailsService

UserDetailsService 作为一个接口，仅仅只做一件事：根据用户名取得对应的用户信息对象，找不到就抛出异常：

```java
public interface UserDetailsService {

	/**
	 * Locates the user based on the username. In the actual implementation, the search
	 * may possibly be case sensitive, or case insensitive depending on how the
	 * implementation instance is configured. In this case, the <code>UserDetails</code>
	 * object that comes back may have a username that is of a different case than what
	 * was actually requested..
	 * @param username the username identifying the user whose data is required.
	 * @return a fully populated user record (never <code>null</code>)
	 * @throws UsernameNotFoundException if the user could not be found or the user has no
	 * GrantedAuthority
	 */
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;

}
```

UserDetailsService 并不关心从哪里找，这是它的实现类需要考虑的事情，可以是内存（比如之前用的 InMemoryUserDetailsManager），可以是关系型数据库，可以是非关系型数据库，可以是远程接口等等。

前面已经演示了从内存中获取用户，这里以一个自定义 UserDetailsService 演示一下，如何构造一个自定义的实现类。

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.security.authentication.InternalAuthenticationServiceException;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.util.Arrays;
import java.util.List;

@Service
public class TextFileUserDetailsService implements UserDetailsService {

    private final Resource usersFile;

    public TextFileUserDetailsService(
            @Value("${demo.security.users-file:classpath:users.txt}")
            Resource usersFile) {
        this.usersFile = usersFile;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        try (
                BufferedReader reader = new BufferedReader(
                        new InputStreamReader(
                                usersFile.getInputStream(),
                                StandardCharsets.UTF_8
                        )
                )
        ) {
            String line;
            int lineNumber = 0;

            while ((line = reader.readLine()) != null) {
                lineNumber++;

                line = line.strip();

                // 跳过空行和注释
                if (line.isEmpty() || line.startsWith("#")) {
                    continue;
                }

                UserDetails user = parseUser(line, lineNumber);

                if (user.getUsername().equals(username)) {
                    return user;
                }
            }
        }
        catch (IOException exception) {
            // 文件无法读取属于认证数据源故障，
            // 不应该伪装成“用户不存在”
            throw new InternalAuthenticationServiceException(
                    "无法读取用户文件：" + usersFile,
                    exception
            );
        }

        throw new UsernameNotFoundException(
                "用户不存在：" + username
        );
    }


    private UserDetails parseUser(String line, int lineNumber) {
        // -1 表示保留末尾的空字符串
        // 例如 Cindy|{noop}123| 的第三段为空，但仍然保留
        String[] fields = line.split("\\|", -1);

        if (fields.length != 3) {
            throw new IllegalStateException(
                    "用户文件第 " + lineNumber
                            + " 行格式错误，应为："
                            + "username|password|authority1,authority2"
            );
        }

        String username = fields[0].strip();
        String password = fields[1].strip();
        String authorityText = fields[2].strip();

        if (username.isEmpty()) {
            throw new IllegalStateException(
                    "用户文件第 " + lineNumber + " 行用户名为空"
            );
        }

        if (password.isEmpty()) {
            throw new IllegalStateException(
                    "用户文件第 " + lineNumber + " 行密码为空"
            );
        }

        List<GrantedAuthority> authorities;

        if (authorityText.isEmpty()) {
            authorities = List.of();
        }
        else {
            authorities = Arrays.stream(authorityText.split(","))
                    .map(String::strip)
                    .filter(value -> !value.isEmpty())
                    .map(SimpleGrantedAuthority::new)
                    .map(authority ->
                            (GrantedAuthority) authority)
                    .toList();
        }

        return User.withUsername(username)
                .password(password)
                .authorities(authorities)
                .build();
    }
}
```

properties 配置文件，指明从哪一个文件读取用户信息：

```
demo.security.users-file=file:C:/Users/ABC/Desktop/users.txt
```

`users.txt` 的格式和内容：

```
Alice|{noop}123|ROLE_ADMIN
Bob|{noop}123|ROLE_USER
Cindy|{noop}123|
```

TextFileUserDetailsService 会从指定的文本文件找到对应的用户信息，并且构建成 User 对象返回。

### UserDetailsManager

UserDetailsService 的职责仅仅是查询用户，而 UserDetailsManager 则是它的扩展：

```java
public interface UserDetailsManager extends UserDetailsService {

	/**
	 * Create a new user with the supplied details.
	 */
	void createUser(UserDetails user);

	/**
	 * Update the specified user.
	 */
	void updateUser(UserDetails user);

	/**
	 * Remove the user with the given login name from the system.
	 */
	void deleteUser(String username);

	/**
	 * Modify the current user's password. This should change the user's password in the
	 * persistent user repository (database, LDAP etc).
	 * @param oldPassword current password (for re-authentication if required)
	 * @param newPassword the password to change to
	 */
	void changePassword(@Nullable String oldPassword, @Nullable String newPassword);

	/**
	 * Check if a user with the supplied login name exists in the system.
	 */
	boolean userExists(String username);

}
```

UserDetailsManager 提供了创建用户、更新用户、删除用户、修改密码、判定用户是否已存在等方法。

常见实现 InMemoryUserDetailsManager、JdbcUserDetailsManager 同时承担认证和用户管理，所以实现的是 UserDetailsManager。
