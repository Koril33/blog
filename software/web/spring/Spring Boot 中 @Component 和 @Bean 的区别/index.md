---
title: "Spring Boot 中 @Component 和 @Bean 的区别"
date: 2026-07-10T10:11:47
summary: "@Component 和 @Bean"
---

## 目录

[TOC]

---

## 区别

`@Bean` 用在第三方框架里的类，特点就是你无法修改第三方的源码，你不能在源码类上面添加注解。

这时候如果想在添加一些自定义代码（一般第三方的类都允许你使用时自定义各种参数来控制生成的类），比如 `RedisClient`：

```java
@Configuration
public class RedisConfig {


    @Bean
    public RedisClient redisClient(){
    	// 这里可以添加自定义的一些配置项代码
    	// 再把配置好的对象返回
        return new RedisClient();
    }

}
```

`@Bean` 作用于方法上，用来告诉 Spring：调用我写的 `@Bean` 注解的方法，帮我管理我返回的这个对象。


`@Component` 往往用在自定义的类，比如：

```java
@Component
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    public String greeting(){
        return "Hello " + userService.getUser() + "!";
    }
}
```

`@Component` 用来类上，Spring 扫描到 `@Component` 注解的类，会自动创建对象，放到容器中管理。

---

## 语义化

`@Component` 属于通用型的注解，如果不知道 `Bean` 处于哪一个业务层次，可以使用 `@Component`。

为了清晰的分层，Spring 还引入了 `@Component` 的语义化注解，这些注解和 `@Component` 本质是一样的：

- `@Controller`：web controller
- `@Service`：业务层
- `@Repository`：数据存储层
- `@Configuration`：配置类

## `@Component` 和 `@Configuration` 的区别

`@Controller` 和 `@Service` 以及 `@Repository` 都是和 `@Component` 没区别，`@Configuration` 虽然也是 `@Component`，但是稍微特殊一些，Spring 扫描到 `@Configuration` 注解的类生成的是代理对象。

`@Configuration` 中的 `@Bean` 如果调用了其他 bean，不会直接 new，而是从容器中找到现有的对象，但是 `@Component` 则会 new 一个新的对象。

考虑一下的三个类，两个普通 pojo，一个配置类：

```java
public class Database {
    
    private final String host;
    private final int port;

    public Database(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public void connect() {
        System.out.println(host + ":" + port);
    }

}

public class MyService {

    private final Database database;

    public MyService(Database database) {
        this.database = database;
    }

    public void doSomething() {
        this.database.connect();
    }
}
```

配置类需要把 `Database` 和 `MyService` 加入到 Spring 容器中管理：

```java
public class MyConfig {

    @Bean
    public Database database() {
        var db = new Database("localhost", 5432);
        System.out.println("MyConfig.database(): " + db);
        return db;
    }

    @Bean
    public MyService myService() {
        var db = database();
        System.out.println("MyConfig.myService(): " + db);
        return new MyService(db);
    }
}
```

所以问题是给 `MyConfig` 添加 `@Component` 和 `@Configuration` 的区别？

如果添加了 `@Component`，那么日志是：

```
MyConfig.database(): site.djhx.demospring.bean.Database@77cb9cd1
MyConfig.database(): site.djhx.demospring.bean.Database@35636217
MyConfig.myService(): site.djhx.demospring.bean.Database@35636217
```

第一条日志是 Spring 在启动的时候需要注册 `Database` `Bean`，创建了 hashcode 为 77cb9cd1 `Database` 对象。

第二条日志是 Spring 注册 `MyService` `Bean` 的时候，执行了 database()，又创建了一个新的 `Database` 对象，hashcode 是 35636217。

可以看到创建 `MyService` 时，依赖的 `Database` 产生了两个对象，这显然不是我们希望的，所以需要改成 `@Configuration` 注解，日志就会变成：

```
MyConfig.database(): site.djhx.demospring.bean.Database@4b869331
MyConfig.myService(): site.djhx.demospring.bean.Database@4b869331
```

只剩两条日志了，是因为在注册 MySerivce `Bean` 时，`MyConfig` 的代理类（`MyConfig$$SpringCGLIB$$0@b25b095`）拦截了 database()，发现容器已经存在 `Database` `Bean`，就不会创建新的 `Database` 对象。

这个例子是为了演示 `@Configuration` 是会生成一个代理类，所以在实例中，我手动 `new Database()`，实际上标准的写法是：

```java
@Bean
public MyService myService(Database db) {
    System.out.println("MyConfig.myService(): " + db);
    return new MyService(db);
}
```

放在函数参数中，而不是函数内部 new，这样依赖关系看的更清楚（`MyService` 依赖 `Database`），Spring 看到函数参数，就会从容器中寻找对应的 bean。

`@Configuration` 的价值不是让 `@Bean` 生效（`@Component` + `@Bean` 也能生效），而是保证配置类内部调用 `@Bean` 注解方法时，返回的是**容器中的 `Bean`，而不是重新创建对象**。

---

## 示例

简单使用各个注解创建一个示例，演示 IOC/DI 的思想。

> IOC（Inversion of Control，控制反转）是思想

>

> DI（Dependency Injection，依赖注入）是实现 IOC 的一种方式。

以简单的 controller -> service -> repository 三层模式为例，没有 Spring IOC 的情况需要在代码手动创建各个依赖对象：

```java
public class UserRepository {

    public String findUser(){
        return "Tom";
    }
}
```

`UserService` 依赖 `UserRepository` 就需要手动在代码里 new：

```java
public class UserService {
    
    private final UserRepository userRepository;
    
    public UserService() {
        this.userRepository = new UserRepository();
    }
    
    public String getUser() {
        return this.userRepository.findUser();
    }
}
```

`UserController` 依赖 `UserService` 也一样：

```java
public class UserController {

    private final UserService userService;

    public UserController() {
    	// 构造器里 new 一个 UserService 对象
        this.userService = new UserService();
    }

    public void greeting() {
        System.out.println("Hello: " + this.userService.getUser());
    }
}
```

上面把创建依赖对象的代码写在了构造器里，如果不在构造器里 new，也得在其他地方 new：

```java
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    public void greeting() {
        System.out.println("Hello: " + this.userService.getUser());
    }

    public static void main(String[] args) {
    	// 手动 new 一个 UserService 对象
        UserService userService = new UserService();
        UserController userController = new UserController(userService);
        userController.greeting();
    }
}
```

如果使用了 Spring，我们只需要声明依赖关系，new 的工作交给了容器：

```java
@Repository
public class UserRepository {


    public String findUser(){
        return "Tom";
    }

}
```

service 依赖于 repository，但我们仅需要通过构造器声明依赖关系，不需要 new：

```java
@Service
public class UserService {


    private final UserRepository repository;


    public UserService(UserRepository repository){
        this.repository = repository;
    }


    public String getUser(){
        return repository.findUser();
    }

}
```

controller 依赖 service：

```java
@Controller
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    public String greeting(){
        return "Hello " + userService.getUser() + "!";
    }
}
```

最后使用一个简单的测试 component 来调用 controller：

```java
@Component
public class TestRunner {
    private final UserController userController;

    public TestRunner(UserController userController) {
        this.userController = userController;
    }

    public void test() {
        System.out.println(userController.greeting());
    }
}
```

`TestRunner` 也通过 `@Component` 被扫描到，进入 Spring 容器：

```java
ConfigurableApplicationContext context = SpringApplication.run(DemoSpringApplication.class, args);

TestRunner testRunner = context.getBean(TestRunner.class);
testRunner.test();
```

以上就是简单的使用方法，我们在原来 pojo 类的基础上简单增加了几个注解，通过构造器声明了几个类的依赖关系，spring 自动实现了扫描，创建类。

---

## IOC 和 DI 的作用？

大型工程往往涉及到非常多的组件调用，IOC 的目标是为了代码的整洁、可维护、可扩展性，实现管理复杂度，解耦合。

考虑到以下代码：

```java
class CheckoutService {
    private final PaymentGateway gateway =
        new StripeGateway(new HttpClient(), env("STRIPE_KEY"));

    private final OrderRepository orders =
        new JdbcOrderRepository(DataSourceFactory.create());
}
```

所有组件手工 new 的好处就是能看的比较明白，所有的逻辑都是“面向过程式”的思维。

但缺点很明显，`CheckoutService` 承担了太多的责任：

1. 选择 Stripe，而不是其他支付实现
2. 创建和配置 HTTP 客户端
3. 读取环境配置
4. 选择 JDBC
5. 创建、共享和关闭数据库资源
6. 手工的组装所有的依赖关系

尤其是通用组件（HTTP 客户端，日志记录器，JDBC，JSON 序列化器等等），如果每个 service 都用到了其中一个或者多个组件，那么每个 service 都会充斥了大量的 new 的代码。

改成 DI 以后，代码就整洁很多：

```java
class CheckoutService {
    private final PaymentGateway gateway;
    private final OrderRepository orders;

    CheckoutService(
        PaymentGateway gateway,
        OrderRepository orders
    ) {
        this.gateway = gateway;
        this.orders = orders;
    }
}
```

`CheckoutService` 仅仅依赖于 `PaymentGateway` 和 `OrderRepository` 两个组件，而且这两个组件都是 Interface，可以有不同的实现类来完成具体的业务功能。

在使用时，依然需要 new：

```java
// new 出具体的实现类
var gateway = new StripeGateway(httpClient, config.stripeKey());
var repository = new JdbcOrderRepository(dataSource);

// 注入到 CheckoutService 中
var checkout = new CheckoutService(gateway, repository);
```

一个 new 都没有少，但创建的责任从业务对象（`CheckoutService`）转移到了另外的地方。 这已经是 DI，不需要 Spring。

Spring 容器只是把对象图解析、作用域和生命周期管理自动化了。

Spring 官方对 DI 的定义也是：对象通过构造器、工厂参数或属性声明依赖，由外部提供，而不是自己构造或查找依赖。

真正的问题一般出现在 new 的是数据库、远程服务、线程池、时钟等长期协作者时：

- 业务类同时依赖 `PaymentGateway` 概念和 `StripeGateway` 细节，更换实现必须修改业务源码。
- 构造器看起来无依赖，实际调用该类的初始化后，会建立数据库连接、读取配置甚至访问网络。
- 底层构造参数变化，所有递归 new 它的上层类都要跟着修改。
- 无法简单传入内存仓库、固定时钟或假支付网关，单测被迫启动真实基础设施。
- 多个地方分别创建 HTTP 客户端、连接池，同一种服务可能使用不同超时和重试策略（我们需要的也许是单例，尤其是基础组件）。
- 每次请求误建连接池，或者资源创建后没人负责关闭。
- 自行 new 的对象可能绕过事务、缓存、监控、鉴权或代理包装。

但也不能所有地方都用 DI，一些 DTO，集合，异常，属于当前业务的内部细节，通常 service 内部直接 new，而外部的共享资源或部署策略的一部分，通常从外部注入。
